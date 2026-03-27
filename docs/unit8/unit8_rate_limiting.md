# Unit 8: Rate Limiting

**Duration**: 60-75 minutes  
**Prerequisites**: Units 1-7 (Go fundamentals, HTTP servers, Authentication, Versioning, Documentation, Caching, Pagination)  
**Goal**: Protect your API from abuse and overload with rate limiting

---

## 8.1 Why Rate Limiting?

### The Problem: API Abuse

Without rate limiting:
```
User sends 10,000 requests/second → Server crashes
Bot scrapes entire database → Data stolen
DDoS attack → Service down for everyone
Buggy client retries infinitely → Resources exhausted
```

**Consequences**:
- 💥 Server overload and crashes
- 💰 Excessive infrastructure costs
- 🐌 Poor performance for all users
- 🔓 Data scraping and theft
- 😡 Frustrated legitimate users

### The Solution: Rate Limiting

With rate limiting:
```
User sends 1,000 requests/second → 100 allowed, 900 rejected
Response: 429 Too Many Requests
Headers: X-RateLimit-Remaining: 0, Retry-After: 60
```

**Benefits**:
- 🛡️ Protects against abuse
- ⚖️ Fair resource allocation
- 💰 Controls costs
- ⚡ Maintains performance
- 🔒 Prevents scraping

---

## 8.2 Rate Limiting Strategies

### Strategy 1: Fixed Window

**Most simple to implement**

```
Window: 1 minute
Limit: 100 requests

[0:00-0:59] → 100 requests allowed
[1:00-1:59] → 100 requests allowed (counter resets)

Request at 0:59 (100th) → Allowed
Request at 1:00 (1st)   → Allowed (new window)
```

**Implementation**:
```go
type FixedWindow struct {
    requests map[string]int  // user -> count
    window   time.Duration
}

func (fw *FixedWindow) Allow(userID string) bool {
    now := time.Now()
    windowStart := now.Truncate(fw.window)
    
    key := fmt.Sprintf("%s:%d", userID, windowStart.Unix())
    
    count := fw.requests[key]
    if count >= 100 {
        return false
    }
    
    fw.requests[key]++
    return true
}
```

**Pros**:
- ✅ Simple to implement
- ✅ Low memory usage
- ✅ Easy to understand

**Cons**:
- ❌ Burst at window boundaries
- ❌ Can allow 2x limit (100 at 0:59, 100 at 1:00)

---

### Strategy 2: Sliding Window Log

**Most accurate**

```
Limit: 100 requests per minute

Track each request timestamp:
[12:00:10, 12:00:15, 12:00:20, ..., 12:00:59]

At 12:01:00:
- Remove requests older than 12:00:00
- Count remaining requests
- Allow if count < 100
```

**Implementation**:
```go
type SlidingWindowLog struct {
    requests map[string][]time.Time
}

func (swl *SlidingWindowLog) Allow(userID string) bool {
    now := time.Now()
    windowStart := now.Add(-1 * time.Minute)
    
    // Get user's requests
    timestamps := swl.requests[userID]
    
    // Remove old requests
    valid := []time.Time{}
    for _, ts := range timestamps {
        if ts.After(windowStart) {
            valid = append(valid, ts)
        }
    }
    
    // Check limit
    if len(valid) >= 100 {
        return false
    }
    
    // Add new request
    valid = append(valid, now)
    swl.requests[userID] = valid
    
    return true
}
```

**Pros**:
- ✅ Very accurate
- ✅ No burst issues
- ✅ Precise enforcement

**Cons**:
- ❌ High memory usage (stores all timestamps)
- ❌ Expensive for high traffic

---

### Strategy 3: Sliding Window Counter

**Best balance of accuracy and efficiency**

```
Combines fixed window simplicity with sliding accuracy

Current window: [1:00-1:59]
Previous window: [0:00-0:59]

At 1:30:
weight = 0.5 (30 seconds into 60-second window)
estimated_count = (prev_count * (1 - weight)) + curr_count
```

**Implementation**:
```go
type SlidingWindowCounter struct {
    current  map[string]int
    previous map[string]int
    lastReset time.Time
}

func (swc *SlidingWindowCounter) Allow(userID string) bool {
    now := time.Now()
    
    // Check if window should rotate
    if now.Sub(swc.lastReset) >= time.Minute {
        swc.previous = swc.current
        swc.current = make(map[string]int)
        swc.lastReset = now.Truncate(time.Minute)
    }
    
    // Calculate sliding count
    elapsed := now.Sub(swc.lastReset)
    weight := float64(elapsed) / float64(time.Minute)
    
    prevCount := swc.previous[userID]
    currCount := swc.current[userID]
    
    estimatedCount := float64(prevCount)*(1-weight) + float64(currCount)
    
    if estimatedCount >= 100 {
        return false
    }
    
    swc.current[userID]++
    return true
}
```

**Pros**:
- ✅ Good accuracy
- ✅ Low memory usage
- ✅ Smooth rate limiting

**Cons**:
- ❌ Slightly complex
- ❌ Approximate (not exact)

---

### Strategy 4: Token Bucket

**Most flexible**

```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/second

Each request consumes 1 token
Tokens refill continuously

Allows bursts up to bucket capacity
Smooth refilling for sustained usage
```

**Implementation**:
```go
type TokenBucket struct {
    capacity    int
    tokens      float64
    refillRate  float64  // tokens per second
    lastRefill  time.Time
}

func (tb *TokenBucket) Allow() bool {
    now := time.Now()
    
    // Refill tokens based on time elapsed
    elapsed := now.Sub(tb.lastRefill).Seconds()
    tb.tokens += elapsed * tb.refillRate
    
    if tb.tokens > float64(tb.capacity) {
        tb.tokens = float64(tb.capacity)
    }
    
    tb.lastRefill = now
    
    // Check if token available
    if tb.tokens < 1 {
        return false
    }
    
    tb.tokens--
    return true
}
```

**Pros**:
- ✅ Allows bursts
- ✅ Smooth long-term rate
- ✅ Flexible

**Cons**:
- ❌ More complex
- ❌ Harder to predict behavior

---

### Strategy 5: Leaky Bucket

**Constant output rate**

```
Requests enter bucket
Process at constant rate (e.g., 100/min)
Excess requests overflow and are rejected

Like water dripping from bucket with hole
```

**Implementation**:
```go
type LeakyBucket struct {
    capacity    int
    queue       []Request
    processRate time.Duration
}

func (lb *LeakyBucket) Allow(req Request) bool {
    // Add to queue if space available
    if len(lb.queue) >= lb.capacity {
        return false
    }
    
    lb.queue = append(lb.queue, req)
    return true
}

// Background processor
func (lb *LeakyBucket) Process() {
    ticker := time.NewTicker(lb.processRate)
    for range ticker.C {
        if len(lb.queue) > 0 {
            req := lb.queue[0]
            lb.queue = lb.queue[1:]
            // Process request
        }
    }
}
```

**Pros**:
- ✅ Constant output rate
- ✅ Smooths traffic
- ✅ Protects backend

**Cons**:
- ❌ Queuing adds latency
- ❌ Complex implementation

---

## 8.3 Implementing Rate Limiting in Go

### Basic Middleware Pattern

```go
type RateLimiter struct {
    limiters map[string]*TokenBucket
    mu       sync.RWMutex
}

func NewRateLimiter() *RateLimiter {
    return &RateLimiter{
        limiters: make(map[string]*TokenBucket),
    }
}

func (rl *RateLimiter) GetLimiter(key string) *TokenBucket {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    limiter, exists := rl.limiters[key]
    if !exists {
        limiter = &TokenBucket{
            capacity:   100,
            tokens:     100,
            refillRate: 10,  // 10 per second
            lastRefill: time.Now(),
        }
        rl.limiters[key] = limiter
    }
    
    return limiter
}

func (rl *RateLimiter) RateLimitMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Get client identifier (IP, user ID, API key)
        clientID := r.RemoteAddr
        
        // Get rate limiter for client
        limiter := rl.GetLimiter(clientID)
        
        // Check if allowed
        if !limiter.Allow() {
            w.Header().Set("X-RateLimit-Limit", "100")
            w.Header().Set("X-RateLimit-Remaining", "0")
            w.Header().Set("Retry-After", "60")
            
            http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
            return
        }
        
        // Set rate limit headers
        w.Header().Set("X-RateLimit-Limit", "100")
        w.Header().Set("X-RateLimit-Remaining", fmt.Sprintf("%.0f", limiter.tokens))
        
        next.ServeHTTP(w, r)
    })
}
```

---

## 8.4 Rate Limit Headers

### Standard Headers

```
X-RateLimit-Limit: 100           # Max requests per window
X-RateLimit-Remaining: 45        # Requests remaining
X-RateLimit-Reset: 1642012800    # Unix timestamp when limit resets
Retry-After: 60                  # Seconds until retry allowed
```

### Usage Example

```go
func setRateLimitHeaders(w http.ResponseWriter, limit, remaining int, reset time.Time) {
    w.Header().Set("X-RateLimit-Limit", strconv.Itoa(limit))
    w.Header().Set("X-RateLimit-Remaining", strconv.Itoa(remaining))
    w.Header().Set("X-RateLimit-Reset", strconv.FormatInt(reset.Unix(), 10))
    
    if remaining == 0 {
        secondsUntilReset := int(time.Until(reset).Seconds())
        w.Header().Set("Retry-After", strconv.Itoa(secondsUntilReset))
    }
}
```

---

## 8.5 Client Identification

### Method 1: IP Address

```go
clientID := r.RemoteAddr  // "192.168.1.1:54321"

// Clean up port
ip := strings.Split(clientID, ":")[0]  // "192.168.1.1"
```

**Pros**: No authentication needed  
**Cons**: 
- Shared IPs (NAT, proxies)
- Easy to bypass (VPN, proxy)

---

### Method 2: API Key

```go
apiKey := r.Header.Get("X-API-Key")
if apiKey == "" {
    http.Error(w, "API key required", http.StatusUnauthorized)
    return
}

clientID := apiKey
```

**Pros**: Accurate per-client  
**Cons**: Requires API key system

---

### Method 3: User ID (Authenticated)

```go
// From JWT or session
userID := r.Context().Value("user_id").(string)

clientID := fmt.Sprintf("user:%s", userID)
```

**Pros**: Per-user limits  
**Cons**: Only for authenticated endpoints

---

### Method 4: Composite Key

```go
// Different limits for different endpoints
endpoint := r.URL.Path
userID := getUserID(r)

clientID := fmt.Sprintf("%s:%s", userID, endpoint)

// Example: "user123:/api/posts" vs "user123:/api/upload"
```

**Allows**: Different rate limits per endpoint

---

## 8.6 Distributed Rate Limiting with Redis

### Redis-Based Rate Limiter

```go
type RedisRateLimiter struct {
    client *redis.Client
    limit  int
    window time.Duration
}

func (rrl *RedisRateLimiter) Allow(clientID string) (bool, int, error) {
    key := fmt.Sprintf("ratelimit:%s", clientID)
    
    // Increment counter
    count, err := rrl.client.Incr(ctx, key).Result()
    if err != nil {
        return false, 0, err
    }
    
    // Set expiration on first request
    if count == 1 {
        rrl.client.Expire(ctx, key, rrl.window)
    }
    
    remaining := rrl.limit - int(count)
    if remaining < 0 {
        remaining = 0
    }
    
    return count <= int64(rrl.limit), remaining, nil
}
```

### Lua Script for Atomic Operations

```go
// Redis Lua script for atomic rate limiting
var rateLimitScript = `
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call('INCR', key)

if current == 1 then
    redis.call('EXPIRE', key, window)
end

if current > limit then
    return {0, 0}  -- not allowed, 0 remaining
end

return {1, limit - current}  -- allowed, remaining count
`

func (rrl *RedisRateLimiter) AllowWithScript(clientID string) (bool, int, error) {
    key := fmt.Sprintf("ratelimit:%s", clientID)
    
    result, err := rrl.client.Eval(
        ctx,
        rateLimitScript,
        []string{key},
        rrl.limit,
        int(rrl.window.Seconds()),
    ).Result()
    
    if err != nil {
        return false, 0, err
    }
    
    values := result.([]interface{})
    allowed := values[0].(int64) == 1
    remaining := int(values[1].(int64))
    
    return allowed, remaining, nil
}
```

**Benefits of Redis**:
- ✅ Shared across multiple servers
- ✅ Atomic operations
- ✅ Automatic expiration
- ✅ High performance

---

## 8.7 Different Limits for Different Users

### Tier-Based Limits

```go
type UserTier string

const (
    TierFree       UserTier = "free"
    TierBasic      UserTier = "basic"
    TierPremium    UserTier = "premium"
    TierEnterprise UserTier = "enterprise"
)

func getLimitForTier(tier UserTier) int {
    limits := map[UserTier]int{
        TierFree:       100,    // 100 req/min
        TierBasic:      1000,   // 1000 req/min
        TierPremium:    10000,  // 10000 req/min
        TierEnterprise: 100000, // 100000 req/min
    }
    
    return limits[tier]
}

func (rl *RateLimiter) RateLimitMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        userID := getUserID(r)
        tier := getUserTier(userID)
        limit := getLimitForTier(tier)
        
        // Check against tier-specific limit
        limiter := rl.GetLimiterWithLimit(userID, limit)
        
        if !limiter.Allow() {
            http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}
```

---

## 8.8 Endpoint-Specific Limits

```go
type EndpointLimits struct {
    "/api/search":     {100, time.Minute},    // Expensive
    "/api/upload":     {10, time.Minute},     // Resource intensive
    "/api/posts":      {1000, time.Minute},   // Normal
    "/api/healthz":    {10000, time.Minute},  // Health checks
}

func (rl *RateLimiter) RateLimitMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        endpoint := r.URL.Path
        userID := getUserID(r)
        
        // Get endpoint-specific limit
        config, exists := EndpointLimits[endpoint]
        if !exists {
            config = DefaultLimit  // Fallback
        }
        
        key := fmt.Sprintf("%s:%s", userID, endpoint)
        limiter := rl.GetLimiterWithConfig(key, config)
        
        if !limiter.Allow() {
            http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}
```

---

## 8.9 Best Practices

### ✅ DO

1. **Return proper HTTP status**
   ```go
   w.WriteHeader(http.StatusTooManyRequests) // 429
   ```

2. **Include helpful headers**
   ```go
   X-RateLimit-Limit
   X-RateLimit-Remaining
   X-RateLimit-Reset
   Retry-After
   ```

3. **Document rate limits**
   - Include in API docs
   - Show limits in responses
   - Explain tier differences

4. **Use distributed rate limiting** (Redis) for multi-server

5. **Different limits for different endpoints**

6. **Graceful degradation**
   ```go
   if rateLimitCheckFails {
       // Allow request but log warning
       // Don't break API if rate limiter is down
   }
   ```

7. **Monitor rate limit hits**
   - Track who's hitting limits
   - Identify abuse patterns
   - Adjust limits based on data

### ❌ DON'T

1. **Don't block health checks**
2. **Don't use same limit for all endpoints**
3. **Don't silently drop requests** (return 429)
4. **Don't forget to test limits**
5. **Don't make limits too restrictive**
6. **Don't trust client-side rate limiting**

---

## 8.10 Complete Example

```go
package main

import (
    "fmt"
    "net/http"
    "sync"
    "time"
)

type TokenBucket struct {
    capacity   float64
    tokens     float64
    refillRate float64
    lastRefill time.Time
    mu         sync.Mutex
}

func NewTokenBucket(capacity, refillRate float64) *TokenBucket {
    return &TokenBucket{
        capacity:   capacity,
        tokens:     capacity,
        refillRate: refillRate,
        lastRefill: time.Now(),
    }
}

func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()
    
    now := time.Now()
    elapsed := now.Sub(tb.lastRefill).Seconds()
    
    tb.tokens += elapsed * tb.refillRate
    if tb.tokens > tb.capacity {
        tb.tokens = tb.capacity
    }
    
    tb.lastRefill = now
    
    if tb.tokens < 1 {
        return false
    }
    
    tb.tokens--
    return true
}

func (tb *TokenBucket) Remaining() int {
    tb.mu.Lock()
    defer tb.mu.Unlock()
    return int(tb.tokens)
}

type RateLimiter struct {
    limiters map[string]*TokenBucket
    mu       sync.RWMutex
}

func NewRateLimiter() *RateLimiter {
    return &RateLimiter{
        limiters: make(map[string]*TokenBucket),
    }
}

func (rl *RateLimiter) GetLimiter(key string) *TokenBucket {
    rl.mu.RLock()
    limiter, exists := rl.limiters[key]
    rl.mu.RUnlock()
    
    if exists {
        return limiter
    }
    
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    limiter = NewTokenBucket(100, 10) // 100 capacity, 10/sec refill
    rl.limiters[key] = limiter
    
    return limiter
}

func (rl *RateLimiter) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        clientID := r.RemoteAddr
        limiter := rl.GetLimiter(clientID)
        
        if !limiter.Allow() {
            w.Header().Set("X-RateLimit-Limit", "100")
            w.Header().Set("X-RateLimit-Remaining", "0")
            w.Header().Set("Retry-After", "10")
            http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
            return
        }
        
        w.Header().Set("X-RateLimit-Limit", "100")
        w.Header().Set("X-RateLimit-Remaining", fmt.Sprintf("%d", limiter.Remaining()))
        
        next.ServeHTTP(w, r)
    })
}

func main() {
    rateLimiter := NewRateLimiter()
    
    http.Handle("/api/", rateLimiter.Middleware(http.HandlerFunc(apiHandler)))
    
    http.ListenAndServe(":8080", nil)
}
```

---

## Key Takeaways

✅ **Rate limiting protects** against abuse and overload  
✅ **Fixed window** is simplest but allows bursts  
✅ **Sliding window** is most accurate but expensive  
✅ **Token bucket** allows bursts with smooth long-term rate  
✅ **Use Redis** for distributed rate limiting  
✅ **Return 429** with helpful headers  
✅ **Different limits** for different users/endpoints  
✅ **Document limits** clearly  
✅ **Monitor** who's hitting limits  

---

## What's Next?

Unit 9 will cover Testing & Integration - writing tests for your complete API! 🧪
