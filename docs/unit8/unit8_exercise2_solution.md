# Unit 8 - Exercise 2 Solution: Distributed Rate Limiting with Redis

**Complete implementation with Redis and sliding window counter**

---

## Full Solution Code

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "strconv"
    "sync/atomic"
    "time"

    "github.com/gorilla/mux"
    "github.com/redis/go-redis/v9"
)

var ctx = context.Background()

// =============================================================================
// MODELS
// =============================================================================

type UserTier string

const (
    TierFree       UserTier = "free"
    TierBasic      UserTier = "basic"
    TierPremium    UserTier = "premium"
    TierEnterprise UserTier = "enterprise"
)

type TierConfig struct {
    Limit  int
    Window time.Duration
}

type RateLimitResult struct {
    Allowed   bool
    Limit     int
    Remaining int
    ResetTime time.Time
}

type RateLimitMetrics struct {
    TotalRequests   int64   `json:"total_requests"`
    LimitedRequests int64   `json:"limited_requests"`
    LimitRate       float64 `json:"limit_rate"`
}

// =============================================================================
// REDIS RATE LIMITER
// =============================================================================

type RedisRateLimiter struct {
    client      *redis.Client
    tierConfigs map[UserTier]TierConfig
    metrics     map[UserTier]*RateLimitMetrics
    windowSize  time.Duration
}

func NewRedisRateLimiter(redisAddr string) *RedisRateLimiter {
    // Connect to Redis
    client := redis.NewClient(&redis.Options{
        Addr:     redisAddr,
        Password: "",
        DB:       0,
    })

    // Test connection
    _, err := client.Ping(ctx).Result()
    if err != nil {
        panic(fmt.Sprintf("Failed to connect to Redis: %v", err))
    }

    // Initialize tier configs
    tierConfigs := map[UserTier]TierConfig{
        TierFree:       {Limit: 100, Window: time.Minute},
        TierBasic:      {Limit: 1000, Window: time.Minute},
        TierPremium:    {Limit: 10000, Window: time.Minute},
        TierEnterprise: {Limit: 100000, Window: time.Minute},
    }

    // Initialize metrics
    metrics := make(map[UserTier]*RateLimitMetrics)
    for tier := range tierConfigs {
        metrics[tier] = &RateLimitMetrics{}
    }

    return &RedisRateLimiter{
        client:      client,
        tierConfigs: tierConfigs,
        metrics:     metrics,
        windowSize:  time.Minute,
    }
}

func (rrl *RedisRateLimiter) CheckRateLimit(userID string, tier UserTier) (RateLimitResult, error) {
    // Get tier config
    config, exists := rrl.tierConfigs[tier]
    if !exists {
        config = rrl.tierConfigs[TierFree] // Default to free
    }

    now := time.Now()
    
    // Calculate current and previous window boundaries
    currentWindow := now.Truncate(rrl.windowSize)
    previousWindow := currentWindow.Add(-rrl.windowSize)

    // Redis keys for current and previous windows
    currentKey := fmt.Sprintf("ratelimit:%s:%s:current:%d", tier, userID, currentWindow.Unix())
    previousKey := fmt.Sprintf("ratelimit:%s:%s:previous:%d", tier, userID, previousWindow.Unix())

    // Get counts from both windows
    pipe := rrl.client.Pipeline()
    currentCmd := pipe.Get(ctx, currentKey)
    previousCmd := pipe.Get(ctx, previousKey)
    _, err := pipe.Exec(ctx)

    // Parse counts (default to 0 if not found)
    currentCount := 0
    previousCount := 0

    if val, err := currentCmd.Result(); err == nil {
        currentCount, _ = strconv.Atoi(val)
    }
    if val, err := previousCmd.Result(); err == nil {
        previousCount, _ = strconv.Atoi(val)
    }

    // Calculate sliding window count
    elapsed := now.Sub(currentWindow)
    weight := float64(elapsed) / float64(rrl.windowSize)
    estimatedCount := int(float64(previousCount)*(1-weight)) + currentCount

    // Check if under limit
    allowed := estimatedCount < config.Limit
    remaining := config.Limit - estimatedCount
    if remaining < 0 {
        remaining = 0
    }

    // Update metrics
    atomic.AddInt64(&rrl.metrics[tier].TotalRequests, 1)
    if !allowed {
        atomic.AddInt64(&rrl.metrics[tier].LimitedRequests, 1)
    }

    // If allowed, increment counter
    if allowed {
        pipe := rrl.client.Pipeline()
        pipe.Incr(ctx, currentKey)
        pipe.Expire(ctx, currentKey, rrl.windowSize*2) // Keep for 2 windows
        pipe.Exec(ctx)
    }

    // Calculate reset time (when current window ends)
    resetTime := currentWindow.Add(rrl.windowSize)

    return RateLimitResult{
        Allowed:   allowed,
        Limit:     config.Limit,
        Remaining: remaining,
        ResetTime: resetTime,
    }, nil
}

func (rrl *RedisRateLimiter) CheckRateLimitWithLua(userID string, tier UserTier) (RateLimitResult, error) {
    // Lua script for atomic sliding window check
    script := `
local current_key = KEYS[1]
local previous_key = KEYS[2]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local current_window_start = tonumber(ARGV[4])

-- Get counts
local current = tonumber(redis.call('GET', current_key) or 0)
local previous = tonumber(redis.call('GET', previous_key) or 0)

-- Calculate sliding window count
local elapsed = now - current_window_start
local weight = elapsed / window
local count = math.floor(previous * (1 - weight)) + current

-- Check limit
if count >= limit then
    return {0, limit, 0}  -- not allowed, limit, 0 remaining
end

-- Increment current counter
redis.call('INCR', current_key)
redis.call('EXPIRE', current_key, window * 2)

local remaining = limit - count - 1
return {1, limit, remaining}  -- allowed, limit, remaining
`

    config := rrl.tierConfigs[tier]
    now := time.Now()
    currentWindow := now.Truncate(rrl.windowSize)
    previousWindow := currentWindow.Add(-rrl.windowSize)

    currentKey := fmt.Sprintf("ratelimit:%s:%s:current:%d", tier, userID, currentWindow.Unix())
    previousKey := fmt.Sprintf("ratelimit:%s:%s:previous:%d", tier, userID, previousWindow.Unix())

    result, err := rrl.client.Eval(
        ctx,
        script,
        []string{currentKey, previousKey},
        config.Limit,
        int(rrl.windowSize.Seconds()),
        now.Unix(),
        currentWindow.Unix(),
    ).Result()

    if err != nil {
        return RateLimitResult{}, err
    }

    values := result.([]interface{})
    allowed := values[0].(int64) == 1
    limit := int(values[1].(int64))
    remaining := int(values[2].(int64))

    resetTime := currentWindow.Add(rrl.windowSize)

    // Update metrics
    atomic.AddInt64(&rrl.metrics[tier].TotalRequests, 1)
    if !allowed {
        atomic.AddInt64(&rrl.metrics[tier].LimitedRequests, 1)
    }

    return RateLimitResult{
        Allowed:   allowed,
        Limit:     limit,
        Remaining: remaining,
        ResetTime: resetTime,
    }, nil
}

func (rrl *RedisRateLimiter) GetMetrics(tier UserTier) RateLimitMetrics {
    metrics := rrl.metrics[tier]
    
    total := atomic.LoadInt64(&metrics.TotalRequests)
    limited := atomic.LoadInt64(&metrics.LimitedRequests)
    
    limitRate := 0.0
    if total > 0 {
        limitRate = float64(limited) / float64(total) * 100
    }
    
    return RateLimitMetrics{
        TotalRequests:   total,
        LimitedRequests: limited,
        LimitRate:       limitRate,
    }
}

func (rrl *RedisRateLimiter) ResetUserLimit(userID string, tier UserTier) error {
    // Delete all Redis keys for this user
    pattern := fmt.Sprintf("ratelimit:%s:%s:*", tier, userID)
    
    iter := rrl.client.Scan(ctx, 0, pattern, 0).Iterator()
    keys := []string{}
    
    for iter.Next(ctx) {
        keys = append(keys, iter.Val())
    }
    
    if len(keys) > 0 {
        return rrl.client.Del(ctx, keys...).Err()
    }
    
    return nil
}

// =============================================================================
// MIDDLEWARE
// =============================================================================

type RateLimitMiddleware struct {
    limiter   *RedisRateLimiter
    whitelist map[string]bool
}

func NewRateLimitMiddleware(limiter *RedisRateLimiter) *RateLimitMiddleware {
    return &RateLimitMiddleware{
        limiter: limiter,
        whitelist: map[string]bool{
            "/api/webhook": true,
            "/api/health":  true,
        },
    }
}

func (rlm *RateLimitMiddleware) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Check if endpoint is whitelisted
        if rlm.whitelist[r.URL.Path] {
            next.ServeHTTP(w, r)
            return
        }

        // Get user ID and tier
        userID, tier := rlm.getUserIDAndTier(r)

        // Check rate limit (using Lua script for atomicity)
        result, err := rlm.limiter.CheckRateLimitWithLua(userID, tier)
        if err != nil {
            // Redis error - fail open (allow request but log)
            fmt.Printf("Rate limit check failed: %v\n", err)
            next.ServeHTTP(w, r)
            return
        }

        // Set rate limit headers
        rlm.setRateLimitHeaders(w, result)

        // If not allowed, return 429
        if !result.Allowed {
            retryAfter := int(time.Until(result.ResetTime).Seconds())
            if retryAfter < 0 {
                retryAfter = 0
            }
            w.Header().Set("Retry-After", fmt.Sprintf("%d", retryAfter))
            
            http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
            return
        }

        // Request allowed
        next.ServeHTTP(w, r)
    })
}

func (rlm *RateLimitMiddleware) getUserIDAndTier(r *http.Request) (string, UserTier) {
    // Try to get from headers (for testing)
    userID := r.Header.Get("X-User-ID")
    tierStr := r.Header.Get("X-User-Tier")

    // Default to IP if not provided
    if userID == "" {
        userID = r.RemoteAddr
    }

    // Parse tier
    tier := TierFree
    switch UserTier(tierStr) {
    case TierBasic:
        tier = TierBasic
    case TierPremium:
        tier = TierPremium
    case TierEnterprise:
        tier = TierEnterprise
    }

    return userID, tier
}

func (rlm *RateLimitMiddleware) setRateLimitHeaders(w http.ResponseWriter, result RateLimitResult) {
    w.Header().Set("X-RateLimit-Limit", strconv.Itoa(result.Limit))
    w.Header().Set("X-RateLimit-Remaining", strconv.Itoa(result.Remaining))
    w.Header().Set("X-RateLimit-Reset", strconv.FormatInt(result.ResetTime.Unix(), 10))
}

// =============================================================================
// HANDLERS
// =============================================================================

type Post struct {
    ID      int    `json:"id"`
    Title   string `json:"title"`
    Content string `json:"content"`
    Author  string `json:"author"`
}

var (
    posts  = []Post{}
    nextID = 1
)

func getPosts(w http.ResponseWriter, r *http.Request) {
    respondJSON(w, http.StatusOK, posts)
}

func createPost(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Title   string `json:"title"`
        Content string `json:"content"`
    }
    
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()
    
    post := Post{
        ID:      nextID,
        Title:   req.Title,
        Content: req.Content,
        Author:  "user",
    }
    posts = append(posts, post)
    nextID++
    
    respondJSON(w, http.StatusCreated, post)
}

func getAnalytics(w http.ResponseWriter, r *http.Request) {
    analytics := map[string]interface{}{
        "total_posts": len(posts),
        "views":       12345,
        "engagement":  0.75,
    }
    
    respondJSON(w, http.StatusOK, analytics)
}

func handleWebhook(w http.ResponseWriter, r *http.Request) {
    respondJSON(w, http.StatusOK, map[string]string{
        "status": "webhook received",
    })
}

func getStats(w http.ResponseWriter, r *http.Request, limiter *RedisRateLimiter) {
    stats := map[UserTier]RateLimitMetrics{
        TierFree:       limiter.GetMetrics(TierFree),
        TierBasic:      limiter.GetMetrics(TierBasic),
        TierPremium:    limiter.GetMetrics(TierPremium),
        TierEnterprise: limiter.GetMetrics(TierEnterprise),
    }
    
    respondJSON(w, http.StatusOK, stats)
}

func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}

// =============================================================================
// MAIN
// =============================================================================

func main() {
    // Initialize Redis rate limiter
    limiter := NewRedisRateLimiter("localhost:6379")
    
    // Create middleware
    rlMiddleware := NewRateLimitMiddleware(limiter)
    
    r := mux.NewRouter()
    
    // Apply middleware
    r.Use(rlMiddleware.Middleware)
    
    // Routes
    r.HandleFunc("/api/posts", getPosts).Methods("GET")
    r.HandleFunc("/api/posts", createPost).Methods("POST")
    r.HandleFunc("/api/analytics", getAnalytics).Methods("GET")
    r.HandleFunc("/api/webhook", handleWebhook).Methods("POST")
    r.HandleFunc("/api/stats", func(w http.ResponseWriter, r *http.Request) {
        getStats(w, r, limiter)
    }).Methods("GET")
    
    fmt.Println("Server starting on :8080")
    fmt.Println("Make sure Redis is running:")
    fmt.Println("  docker run -d -p 6379:6379 redis:latest")
    fmt.Println("\nTier limits:")
    fmt.Println("  Free:       100 requests/min")
    fmt.Println("  Basic:      1,000 requests/min")
    fmt.Println("  Premium:    10,000 requests/min")
    fmt.Println("  Enterprise: 100,000 requests/min")
    fmt.Println("\nTest with headers:")
    fmt.Println("  curl -H 'X-User-ID: user1' -H 'X-User-Tier: free' http://localhost:8080/api/posts")
    
    http.ListenAndServe(":8080", r)
}
```

---

## Key Concepts Explained

### 1. Sliding Window Counter Algorithm

```go
// Current window: [12:00 - 12:01]
// Previous window: [11:59 - 12:00]
// Now: 12:00:30 (30 seconds into current window)

previousCount := 80  // requests in previous window
currentCount := 40   // requests in current window

// Calculate weight (how far into current window)
elapsed := 30 seconds
weight := 30 / 60 = 0.5

// Sliding count = weighted average
estimatedCount = previous * (1 - weight) + current
               = 80 * 0.5 + 40
               = 40 + 40
               = 80

// Check limit
if estimatedCount < 100 {
    allow()
}
```

**Benefits**:
- More accurate than fixed window
- Less memory than sliding log
- Prevents burst at window boundaries

### 2. Distributed Rate Limiting with Redis

```go
// Redis stores counts across all servers
Server 1: increments "ratelimit:free:user1:current:1642012800"
Server 2: reads same key, sees updated count
Server 3: reads same key, sees updated count

// All servers see same count → consistent limit
```

**Without Redis** (in-memory only):
```
Server 1: user1 → 50 requests (under 100)
Server 2: user1 → 50 requests (under 100)
Server 3: user1 → 50 requests (under 100)
Total: 150 requests (exceeded 100!) ❌
```

**With Redis**:
```
All servers share Redis:
Total count: 150 requests → 50 allowed, 100 denied ✓
```

### 3. Atomic Operations with Lua Script

```lua
-- All operations in this script are atomic
local current = redis.call('GET', current_key)
local previous = redis.call('GET', previous_key)
local count = calculate_sliding(current, previous)

if count >= limit then
    return {0}  -- denied
end

redis.call('INCR', current_key)
return {1}  -- allowed
```

**Why Lua?**:
- All operations execute atomically
- No race conditions between requests
- Prevents: check-then-act race

**Without Lua** (potential race):
```go
// Request A: reads count = 99
// Request B: reads count = 99
// Request A: increments to 100 → allowed
// Request B: increments to 101 → allowed (should be denied!)
```

### 4. Tier-Based Limits

```go
tierConfigs := map[UserTier]TierConfig{
    TierFree:       {Limit: 100, Window: time.Minute},
    TierBasic:      {Limit: 1000, Window: time.Minute},
    TierPremium:    {Limit: 10000, Window: time.Minute},
    TierEnterprise: {Limit: 100000, Window: time.Minute},
}

// Different Redis keys per tier
"ratelimit:free:user1:current:..."
"ratelimit:premium:user2:current:..."
```

**Benefit**: Users pay for higher limits

### 5. Graceful Degradation

```go
result, err := limiter.CheckRateLimit(userID, tier)
if err != nil {
    // Redis is down - fail open
    // Allow request but log warning
    log.Warn("Redis unavailable, allowing request")
    next.ServeHTTP(w, r)
    return
}
```

**Philosophy**: Better to allow requests than break API when Redis fails

---

## Testing the Solution

### Test 1: Basic Rate Limiting

```bash
# Start Redis
docker run -d -p 6379:6379 redis:latest

# Free tier (100 limit)
for i in {1..120}; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}\n" \
    -H "X-User-ID: user1" \
    -H "X-User-Tier: free" \
    http://localhost:8080/api/posts)
  echo "$i: $STATUS"
done
```

**Expected**: 100× 200, then 20× 429

### Test 2: Distributed Consistency

```bash
# Start two servers
# Terminal 1
go run main.go

# Terminal 2
PORT=8081 go run main.go

# Terminal 3: Send requests to both
for i in {1..60}; do
  curl -s -H "X-User-ID: user2" -H "X-User-Tier: free" \
    http://localhost:8080/api/posts > /dev/null
done

for i in {1..60}; do
  curl -s -H "X-User-ID: user2" -H "X-User-Tier: free" \
    http://localhost:8081/api/posts > /dev/null
done

# Both servers see same Redis count
# Total: 120 requests → 100 allowed, 20 denied
```

### Test 3: Sliding Window

```bash
# At :00, use 100 requests
for i in {1..100}; do
  curl -s -H "X-User-ID: user3" -H "X-User-Tier: free" \
    http://localhost:8080/api/posts > /dev/null
done

# At :30, sliding window count = 100 * 0.5 + 0 = 50
# Can make 50 more requests

sleep 30

for i in {1..60}; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}\n" \
    -H "X-User-ID: user3" -H "X-User-Tier: free" \
    http://localhost:8080/api/posts)
  echo "$i: $STATUS"
done

# Expected: ~50× 200, then 10× 429
```

---

## What You've Learned

✅ **Redis-based distributed rate limiting**  
✅ **Sliding window counter** algorithm  
✅ **Atomic operations** with Lua scripts  
✅ **Tier-based limits** for different user levels  
✅ **Graceful degradation** when Redis fails  
✅ **Metrics tracking** across tiers  
✅ **Whitelist** for unlimited endpoints  
✅ **Distributed consistency** across multiple servers  

You now understand production-grade distributed rate limiting used by Stripe, GitHub, and Twitter! 🚀
