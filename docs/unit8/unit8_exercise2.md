# Unit 8 - Exercise 2: Distributed Rate Limiting with Redis

**Difficulty**: Advanced  
**Estimated Time**: 60-75 minutes  
**Concepts Covered**: Redis rate limiting, sliding window counter, distributed systems, Lua scripts, tier-based limits

---

## Objective

Implement a distributed rate limiter using Redis that:
- Works across multiple API servers
- Uses sliding window counter algorithm
- Supports tier-based limits (Free, Basic, Premium)
- Uses Lua scripts for atomic operations
- Tracks rate limit metrics
- Handles Redis failures gracefully

---

## Requirements

### User Tiers

| Tier | Limit | Window |
|------|-------|--------|
| Free | 100 requests | 1 minute |
| Basic | 1,000 requests | 1 minute |
| Premium | 10,000 requests | 1 minute |
| Enterprise | 100,000 requests | 1 minute |

### Endpoints

| Method | Path | Description | Tier Required |
|--------|------|-------------|---------------|
| GET | /api/posts | List posts | Any |
| POST | /api/posts | Create post | Basic+ |
| GET | /api/analytics | View analytics | Premium+ |
| POST | /api/webhook | Webhook endpoint | No limit (whitelisted) |
| GET | /api/stats | Rate limit stats | Any |

### Redis Keys

```
ratelimit:{tier}:{user_id}:current   # Current window count
ratelimit:{tier}:{user_id}:previous  # Previous window count
ratelimit:metrics:{tier}             # Tier metrics
```

---

## Starter Code

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
    TotalRequests   int64 `json:"total_requests"`
    LimitedRequests int64 `json:"limited_requests"`
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

// TODO: Implement NewRedisRateLimiter
func NewRedisRateLimiter(redisAddr string) *RedisRateLimiter {
    // Connect to Redis
    // Initialize tier configs
    // Initialize metrics tracking
}

// TODO: Implement CheckRateLimit with sliding window counter
func (rrl *RedisRateLimiter) CheckRateLimit(userID string, tier UserTier) (RateLimitResult, error) {
    // 1. Get tier config
    // 2. Get current timestamp and window boundaries
    // 3. Get counts from current and previous windows
    // 4. Calculate sliding window count
    // 5. Check if under limit
    // 6. Increment current window counter
    // 7. Set expiration
    // 8. Update metrics
    // 9. Return result
}

// TODO: Implement CheckRateLimitWithLua (Bonus)
func (rrl *RedisRateLimiter) CheckRateLimitWithLua(userID string, tier UserTier) (RateLimitResult, error) {
    // Use Lua script for atomic sliding window check
}

// TODO: Implement GetMetrics
func (rrl *RedisRateLimiter) GetMetrics(tier UserTier) RateLimitMetrics {
    // Return metrics for tier
}

// TODO: Implement ResetUserLimit (for testing)
func (rrl *RedisRateLimiter) ResetUserLimit(userID string, tier UserTier) error {
    // Delete Redis keys for user
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

// TODO: Implement Middleware
func (rlm *RateLimitMiddleware) Middleware(next http.Handler) http.Handler {
    // 1. Check if endpoint is whitelisted
    // 2. Get user ID and tier from request
    // 3. Check rate limit
    // 4. If limited:
    //    - Set headers
    //    - Return 429
    // 5. If allowed:
    //    - Set headers
    //    - Call next handler
}

// TODO: Implement getUserIDAndTier
func (rlm *RateLimitMiddleware) getUserIDAndTier(r *http.Request) (string, UserTier) {
    // Extract from headers, JWT, or session
    // For exercise: use X-User-ID and X-User-Tier headers
    // Fallback to IP and Free tier
}

// TODO: Implement setRateLimitHeaders
func (rlm *RateLimitMiddleware) setRateLimitHeaders(w http.ResponseWriter, result RateLimitResult) {
    // Set standard rate limit headers
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
    // Premium feature
    analytics := map[string]interface{}{
        "total_posts": len(posts),
        "views":       12345,
        "engagement":  0.75,
    }
    
    respondJSON(w, http.StatusOK, analytics)
}

func handleWebhook(w http.ResponseWriter, r *http.Request) {
    // No rate limit (whitelisted)
    respondJSON(w, http.StatusOK, map[string]string{
        "status": "webhook received",
    })
}

func getStats(w http.ResponseWriter, r *http.Request) {
    // Get rate limit metrics
    // TODO: Return metrics for all tiers
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
    // TODO: Initialize Redis rate limiter
    // limiter := NewRedisRateLimiter("localhost:6379")
    
    // TODO: Create middleware
    // rlMiddleware := NewRateLimitMiddleware(limiter)
    
    r := mux.NewRouter()
    
    // TODO: Apply middleware
    
    // Routes
    r.HandleFunc("/api/posts", getPosts).Methods("GET")
    r.HandleFunc("/api/posts", createPost).Methods("POST")
    r.HandleFunc("/api/analytics", getAnalytics).Methods("GET")
    r.HandleFunc("/api/webhook", handleWebhook).Methods("POST")
    r.HandleFunc("/api/stats", getStats).Methods("GET")
    
    fmt.Println("Server starting on :8080")
    fmt.Println("Make sure Redis is running: docker run -d -p 6379:6379 redis:latest")
    fmt.Println("\nTier limits:")
    fmt.Println("  Free:       100 requests/min")
    fmt.Println("  Basic:      1,000 requests/min")
    fmt.Println("  Premium:    10,000 requests/min")
    fmt.Println("  Enterprise: 100,000 requests/min")
    
    http.ListenAndServe(":8080", r)
}
```

---

## Your Tasks

### Task 1: Implement Redis Connection

Create Redis client and tier configurations:
```go
tierConfigs := map[UserTier]TierConfig{
    TierFree:       {Limit: 100, Window: time.Minute},
    TierBasic:      {Limit: 1000, Window: time.Minute},
    TierPremium:    {Limit: 10000, Window: time.Minute},
    TierEnterprise: {Limit: 100000, Window: time.Minute},
}
```

### Task 2: Implement Sliding Window Counter

Use Redis to implement sliding window:
1. Get current and previous window counts
2. Calculate weighted average:
   ```
   weight = time_into_current_window / window_size
   count = prev * (1 - weight) + current
   ```
3. Check if count < limit
4. Increment current counter

### Task 3: Implement Atomic Operations

Make rate limit check atomic:
- Get previous and current counts
- Increment current count
- All in one operation (or use Lua script)

### Task 4: Implement Middleware

Create middleware that:
- Extracts user ID and tier
- Checks whitelist
- Calls rate limiter
- Sets headers
- Returns 429 or allows request

### Task 5: Track Metrics

For each tier, track:
- Total requests
- Limited requests  
- Limit rate (percentage)

### Task 6: Implement Lua Script (Bonus)

Create Lua script for atomic sliding window check:
```lua
local current_key = KEYS[1]
local previous_key = KEYS[2]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

-- Get counts
local current = tonumber(redis.call('GET', current_key) or 0)
local previous = tonumber(redis.call('GET', previous_key) or 0)

-- Calculate sliding count
local now = redis.call('TIME')
local elapsed = now[1] % window
local weight = elapsed / window
local count = previous * (1 - weight) + current

-- Check limit
if count >= limit then
    return {0, limit, 0}  -- not allowed
end

-- Increment
redis.call('INCR', current_key)
redis.call('EXPIRE', current_key, window * 2)

return {1, limit, limit - count - 1}  -- allowed
```

---

## Testing Your Implementation

### Test 1: Basic Rate Limiting

```bash
# Free tier user
for i in {1..120}; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}\n" \
    -H "X-User-ID: user1" \
    -H "X-User-Tier: free" \
    http://localhost:8080/api/posts)
  echo "Request $i: $STATUS"
done

# First 100 should be 200
# Next 20 should be 429
```

### Test 2: Tier Differences

```bash
# Free user (100 limit)
for i in {1..150}; do
  curl -s -H "X-User-ID: user1" -H "X-User-Tier: free" \
    http://localhost:8080/api/posts > /dev/null
done

# Basic user (1000 limit)
for i in {1..150}; do
  curl -s -H "X-User-ID: user2" -H "X-User-Tier: basic" \
    http://localhost:8080/api/posts > /dev/null
done

# Free user should be limited
# Basic user should succeed
```

### Test 3: Sliding Window

```bash
# Use 100 requests at end of minute
sleep 50  # Wait until :50 seconds
for i in {1..100}; do
  curl -s -H "X-User-ID: user3" -H "X-User-Tier: free" \
    http://localhost:8080/api/posts > /dev/null
done

# Wait 20 seconds (now at :10 of next minute)
sleep 20

# Can make ~20 more requests (sliding window)
for i in {1..30}; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}\n" \
    -H "X-User-ID: user3" -H "X-User-Tier: free" \
    http://localhost:8080/api/posts)
  echo "Request $i: $STATUS"
done
```

### Test 4: Distributed (Multiple Servers)

```bash
# Terminal 1: Start server 1
go run main.go

# Terminal 2: Start server 2 on different port
PORT=8081 go run main.go

# Terminal 3: Make requests to both servers
for i in {1..60}; do
  curl -s -H "X-User-ID: user4" -H "X-User-Tier: free" \
    http://localhost:8080/api/posts > /dev/null
done

for i in {1..60}; do
  curl -s -H "X-User-ID: user4" -H "X-User-Tier: free" \
    http://localhost:8081/api/posts > /dev/null
done

# Total should still respect 100 limit across both servers
```

### Test 5: Whitelist

```bash
# Webhook endpoint should not be rate limited
for i in {1..200}; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}\n" \
    -X POST http://localhost:8080/api/webhook)
  echo "Request $i: $STATUS"
done

# All should be 200
```

---

## Expected Behavior

### Free Tier Request
```
GET /api/posts
Headers:
  X-User-ID: user1
  X-User-Tier: free

Response: 200 OK
Headers:
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 95
  X-RateLimit-Reset: 1642012860
```

### After Exceeding Limit
```
GET /api/posts
Headers:
  X-User-ID: user1
  X-User-Tier: free

Response: 429 Too Many Requests
Headers:
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 0
  X-RateLimit-Reset: 1642012920
  Retry-After: 60
Body:
  {"error": "Rate limit exceeded"}
```

### Premium Tier
```
GET /api/posts
Headers:
  X-User-ID: user2
  X-User-Tier: premium

Response: 200 OK
Headers:
  X-RateLimit-Limit: 10000
  X-RateLimit-Remaining: 9995
```

---

## Bonus Challenges

### Bonus 1: Graceful Degradation

Handle Redis failures:
```go
func (rrl *RedisRateLimiter) CheckRateLimit(...) (RateLimitResult, error) {
    // Try Redis
    result, err := rrl.checkRedis(...)
    if err != nil {
        // Redis down - allow request but log warning
        log.Warn("Redis unavailable, allowing request")
        return RateLimitResult{Allowed: true}, nil
    }
    return result, nil
}
```

### Bonus 2: Cost-Based Limiting

Different endpoints consume different amounts:
```go
type EndpointCost map[string]int

costs := EndpointCost{
    "/api/posts":     1,
    "/api/analytics": 10,
    "/api/export":    100,
}
```

### Bonus 3: Burst Allowance

Allow short bursts above limit:
```go
type TierConfig struct {
    Limit       int
    BurstLimit  int  // e.g., 120 for Free (20% burst)
    Window      time.Duration
}
```

### Bonus 4: Rate Limit by IP + User

Combine IP and user ID for rate limiting:
```go
key := fmt.Sprintf("%s:%s", userID, ip)
```

### Bonus 5: Dashboard Endpoint

Show real-time metrics:
```go
GET /api/admin/rate-limits

{
  "free": {
    "total_requests": 15234,
    "limited_requests": 523,
    "limit_rate": 3.4
  },
  "premium": {...}
}
```

---

## Hints

### Hint 1: Sliding Window Calculation

```go
now := time.Now()
windowStart := now.Truncate(rrl.windowSize)
prevWindowStart := windowStart.Add(-rrl.windowSize)

currentKey := fmt.Sprintf("ratelimit:%s:%s:current", tier, userID)
previousKey := fmt.Sprintf("ratelimit:%s:%s:previous", tier, userID)

// Get counts
current, _ := rrl.client.Get(ctx, currentKey).Int()
previous, _ := rrl.client.Get(ctx, previousKey).Int()

// Calculate weight
elapsed := now.Sub(windowStart)
weight := float64(elapsed) / float64(rrl.windowSize)

// Sliding count
count := int(float64(previous)*(1-weight)) + current
```

### Hint 2: Window Rotation

```go
// Check if we need to rotate windows
if now.Sub(windowStart) >= rrl.windowSize {
    // Rotate: current becomes previous
    pipe := rrl.client.Pipeline()
    pipe.Rename(ctx, currentKey, previousKey)
    pipe.Del(ctx, currentKey)
    pipe.Exec(ctx)
}
```

### Hint 3: Atomic Increment

```go
// Increment and set expiration atomically
pipe := rrl.client.Pipeline()
pipe.Incr(ctx, currentKey)
pipe.Expire(ctx, currentKey, rrl.windowSize*2)
pipe.Exec(ctx)
```

---

## What You're Learning

✅ **Redis-based rate limiting** for distributed systems  
✅ **Sliding window counter** algorithm  
✅ **Atomic operations** with Redis  
✅ **Lua scripts** for complex Redis operations  
✅ **Tier-based limits** for different user levels  
✅ **Graceful degradation** when Redis fails  
✅ **Metrics tracking** for monitoring  
✅ **Distributed consistency** across servers  

This is production-grade rate limiting used by major APIs!
