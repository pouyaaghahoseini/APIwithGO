# Unit 8 - Exercise 1 Solution: Token Bucket Rate Limiter

**Complete implementation with explanations**

---

## Full Solution Code

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strings"
    "sync"
    "time"

    "github.com/gorilla/mux"
)

// =============================================================================
// TOKEN BUCKET
// =============================================================================

type TokenBucket struct {
    capacity   float64
    tokens     float64
    refillRate float64 // tokens per second
    lastRefill time.Time
    mu         sync.Mutex
}

func NewTokenBucket(capacity, refillRate float64) *TokenBucket {
    return &TokenBucket{
        capacity:   capacity,
        tokens:     capacity, // Start with full bucket
        refillRate: refillRate,
        lastRefill: time.Now(),
    }
}

func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    now := time.Now()
    
    // Calculate how much time has passed
    elapsed := now.Sub(tb.lastRefill).Seconds()
    
    // Add tokens based on elapsed time and refill rate
    tb.tokens += elapsed * tb.refillRate
    
    // Cap tokens at capacity
    if tb.tokens > tb.capacity {
        tb.tokens = tb.capacity
    }
    
    // Update last refill time
    tb.lastRefill = now
    
    // Check if we have at least 1 token
    if tb.tokens < 1 {
        return false
    }
    
    // Consume 1 token
    tb.tokens--
    return true
}

func (tb *TokenBucket) Remaining() int {
    tb.mu.Lock()
    defer tb.mu.Unlock()
    
    // Return floor of current tokens
    return int(tb.tokens)
}

func (tb *TokenBucket) ResetTime() time.Time {
    tb.mu.Lock()
    defer tb.mu.Unlock()
    
    // If bucket is full or more, reset time is now
    if tb.tokens >= tb.capacity {
        return time.Now()
    }
    
    // Calculate how many tokens needed to fill
    tokensNeeded := tb.capacity - tb.tokens
    
    // Calculate seconds needed at current refill rate
    secondsToFill := tokensNeeded / tb.refillRate
    
    // Return time when bucket will be full
    return time.Now().Add(time.Duration(secondsToFill) * time.Second)
}

// =============================================================================
// RATE LIMITER
// =============================================================================

type RateLimiter struct {
    limiters       map[string]*TokenBucket
    mu             sync.RWMutex
    cleanupTicker  *time.Ticker
    defaultConfig  LimitConfig
    endpointLimits map[string]LimitConfig
    lastAccess     map[string]time.Time
}

type LimitConfig struct {
    Capacity   float64
    RefillRate float64
}

func NewRateLimiter(defaultConfig LimitConfig) *RateLimiter {
    rl := &RateLimiter{
        limiters:       make(map[string]*TokenBucket),
        defaultConfig:  defaultConfig,
        endpointLimits: make(map[string]LimitConfig),
        lastAccess:     make(map[string]time.Time),
    }
    
    // Start cleanup goroutine
    rl.startCleanup()
    
    return rl
}

func (rl *RateLimiter) GetLimiter(clientID string, config LimitConfig) *TokenBucket {
    // Try to get existing limiter (read lock)
    rl.mu.RLock()
    limiter, exists := rl.limiters[clientID]
    rl.mu.RUnlock()
    
    if exists {
        // Update last access time
        rl.mu.Lock()
        rl.lastAccess[clientID] = time.Now()
        rl.mu.Unlock()
        return limiter
    }
    
    // Create new limiter (write lock)
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    // Double-check in case another goroutine created it
    limiter, exists = rl.limiters[clientID]
    if exists {
        rl.lastAccess[clientID] = time.Now()
        return limiter
    }
    
    // Create new token bucket
    limiter = NewTokenBucket(config.Capacity, config.RefillRate)
    rl.limiters[clientID] = limiter
    rl.lastAccess[clientID] = time.Now()
    
    return limiter
}

func (rl *RateLimiter) getClientID(r *http.Request) string {
    // Get IP from RemoteAddr
    ip := r.RemoteAddr
    
    // Remove port number (format is "IP:port")
    if idx := strings.LastIndex(ip, ":"); idx != -1 {
        ip = ip[:idx]
    }
    
    return ip
}

func (rl *RateLimiter) getLimitConfig(endpoint string) LimitConfig {
    // Check if endpoint has specific config
    if config, exists := rl.endpointLimits[endpoint]; exists {
        return config
    }
    
    // Return default config
    return rl.defaultConfig
}

func (rl *RateLimiter) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Get client identifier
        clientID := rl.getClientID(r)
        
        // Get limit config for this endpoint
        endpoint := r.URL.Path
        config := rl.getLimitConfig(endpoint)
        
        // Create unique key for this client + endpoint
        key := fmt.Sprintf("%s:%s", clientID, endpoint)
        
        // Get or create rate limiter for this client
        limiter := rl.GetLimiter(key, config)
        
        // Check if request is allowed
        if !limiter.Allow() {
            // Rate limit exceeded
            remaining := limiter.Remaining()
            resetTime := limiter.ResetTime()
            
            // Set rate limit headers
            w.Header().Set("X-RateLimit-Limit", fmt.Sprintf("%.0f", config.Capacity))
            w.Header().Set("X-RateLimit-Remaining", "0")
            w.Header().Set("X-RateLimit-Reset", fmt.Sprintf("%d", resetTime.Unix()))
            
            // Calculate retry-after in seconds
            retryAfter := int(time.Until(resetTime).Seconds())
            if retryAfter < 0 {
                retryAfter = 0
            }
            w.Header().Set("Retry-After", fmt.Sprintf("%d", retryAfter))
            
            // Return 429 Too Many Requests
            http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
            return
        }
        
        // Request allowed - set headers
        remaining := limiter.Remaining()
        resetTime := limiter.ResetTime()
        
        w.Header().Set("X-RateLimit-Limit", fmt.Sprintf("%.0f", config.Capacity))
        w.Header().Set("X-RateLimit-Remaining", fmt.Sprintf("%d", remaining))
        w.Header().Set("X-RateLimit-Reset", fmt.Sprintf("%d", resetTime.Unix()))
        
        // Call next handler
        next.ServeHTTP(w, r)
    })
}

func (rl *RateLimiter) cleanup() {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    // Remove limiters that haven't been accessed in 5 minutes
    threshold := time.Now().Add(-5 * time.Minute)
    
    for key, lastAccess := range rl.lastAccess {
        if lastAccess.Before(threshold) {
            delete(rl.limiters, key)
            delete(rl.lastAccess, key)
        }
    }
}

func (rl *RateLimiter) startCleanup() {
    // Create ticker for cleanup
    rl.cleanupTicker = time.NewTicker(1 * time.Minute)
    
    // Start cleanup goroutine
    go func() {
        for range rl.cleanupTicker.C {
            rl.cleanup()
        }
    }()
}

func (rl *RateLimiter) Stop() {
    if rl.cleanupTicker != nil {
        rl.cleanupTicker.Stop()
    }
}

// =============================================================================
// HANDLERS
// =============================================================================

type Post struct {
    ID      int    `json:"id"`
    Title   string `json:"title"`
    Content string `json:"content"`
}

var (
    posts   = []Post{}
    postsMu sync.RWMutex
    nextID  = 1
)

func getPosts(w http.ResponseWriter, r *http.Request) {
    postsMu.RLock()
    defer postsMu.RUnlock()
    
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
    
    postsMu.Lock()
    post := Post{
        ID:      nextID,
        Title:   req.Title,
        Content: req.Content,
    }
    posts = append(posts, post)
    nextID++
    postsMu.Unlock()
    
    respondJSON(w, http.StatusCreated, post)
}

func uploadFile(w http.ResponseWriter, r *http.Request) {
    // Simulate expensive operation
    time.Sleep(100 * time.Millisecond)
    
    respondJSON(w, http.StatusOK, map[string]string{
        "message": "File uploaded successfully",
    })
}

func searchPosts(w http.ResponseWriter, r *http.Request) {
    query := r.URL.Query().Get("q")
    
    // Simulate expensive search
    time.Sleep(50 * time.Millisecond)
    
    respondJSON(w, http.StatusOK, map[string]string{
        "query":   query,
        "results": "Search results here",
    })
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
    // Create rate limiter with default config
    rateLimiter := NewRateLimiter(LimitConfig{
        Capacity:   100,
        RefillRate: 10, // 10 tokens per second
    })
    
    // Set endpoint-specific limits
    rateLimiter.endpointLimits = map[string]LimitConfig{
        "/api/upload": {
            Capacity:   10,
            RefillRate: 1, // 1 token per second
        },
        "/api/search": {
            Capacity:   50,
            RefillRate: 5, // 5 tokens per second
        },
    }
    
    r := mux.NewRouter()
    
    // Apply rate limiter middleware to all routes
    r.Use(rateLimiter.Middleware)
    
    // Routes
    r.HandleFunc("/api/posts", getPosts).Methods("GET")
    r.HandleFunc("/api/posts", createPost).Methods("POST")
    r.HandleFunc("/api/upload", uploadFile).Methods("POST")
    r.HandleFunc("/api/search", searchPosts).Methods("GET")
    
    fmt.Println("Server starting on :8080")
    fmt.Println("\nRate limits:")
    fmt.Println("  /api/posts  - 100 capacity, 10 tokens/sec refill")
    fmt.Println("  /api/upload - 10 capacity, 1 token/sec refill")
    fmt.Println("  /api/search - 50 capacity, 5 tokens/sec refill")
    fmt.Println("\nExample:")
    fmt.Println("  curl -i http://localhost:8080/api/posts")
    
    http.ListenAndServe(":8080", r)
}
```

---

## Key Concepts Explained

### 1. Token Bucket Algorithm

```go
func (tb *TokenBucket) Allow() bool {
    // Calculate elapsed time
    now := time.Now()
    elapsed := now.Sub(tb.lastRefill).Seconds()
    
    // Refill tokens: tokens += time * rate
    tb.tokens += elapsed * tb.refillRate
    
    // Cap at capacity
    if tb.tokens > tb.capacity {
        tb.tokens = tb.capacity
    }
    
    // Update timestamp
    tb.lastRefill = now
    
    // Check availability
    if tb.tokens < 1 {
        return false  // No tokens available
    }
    
    // Consume token
    tb.tokens--
    return true
}
```

**How it works**:
1. **Bucket** holds tokens (e.g., 100 capacity)
2. **Refilling** happens continuously (e.g., 10 tokens/second)
3. **Request** consumes 1 token
4. **Allow** if tokens available, **deny** if empty

**Example timeline**:
```
Time 0s:  100 tokens (full)
Request:  99 tokens (consumed 1)
Request:  98 tokens
...
Time 10s: 98 + (10 * 10) = 198 → capped at 100
Request:  99 tokens
```

### 2. Mutex for Thread Safety

```go
func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()
    
    // Critical section - only one goroutine at a time
    // Prevents race conditions when:
    // - Reading tokens
    // - Updating tokens
    // - Modifying lastRefill
}
```

**Why needed**: Multiple requests can arrive simultaneously, need to prevent:
- Two requests reading same token count
- Race condition on token consumption
- Inconsistent state

### 3. Reset Time Calculation

```go
func (tb *TokenBucket) ResetTime() time.Time {
    if tb.tokens >= tb.capacity {
        return time.Now()  // Already full
    }
    
    tokensNeeded := tb.capacity - tb.tokens
    secondsToFill := tokensNeeded / tb.refillRate
    
    return time.Now().Add(time.Duration(secondsToFill) * time.Second)
}
```

**Example**:
```
Current tokens: 40
Capacity: 100
Refill rate: 10/sec

Tokens needed: 100 - 40 = 60
Time to fill: 60 / 10 = 6 seconds
Reset time: now + 6 seconds
```

### 4. Per-Client, Per-Endpoint Rate Limiting

```go
func (rl *RateLimiter) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        clientID := rl.getClientID(r)       // "192.168.1.1"
        endpoint := r.URL.Path               // "/api/upload"
        config := rl.getLimitConfig(endpoint) // {10, 1}
        
        key := fmt.Sprintf("%s:%s", clientID, endpoint)
        // "192.168.1.1:/api/upload"
        
        limiter := rl.GetLimiter(key, config)
        // Each client has separate limit per endpoint
    })
}
```

**Benefits**:
- Client A hitting `/api/posts` doesn't affect their `/api/upload` limit
- Client A and Client B have separate limits
- Different endpoints can have different limits

### 5. Cleanup to Prevent Memory Leaks

```go
func (rl *RateLimiter) cleanup() {
    threshold := time.Now().Add(-5 * time.Minute)
    
    for key, lastAccess := range rl.lastAccess {
        if lastAccess.Before(threshold) {
            delete(rl.limiters, key)
            delete(rl.lastAccess, key)
        }
    }
}
```

**Why needed**:
- Every unique client creates a limiter
- Without cleanup, memory grows unbounded
- Remove limiters inactive for 5+ minutes
- Run cleanup every 1 minute

**Example**:
```
Time 0:00 - Client 1 makes request → limiter created
Time 0:30 - Client 2 makes request → limiter created
Time 6:00 - Cleanup runs
            - Client 1 last access: 6 min ago → removed
            - Client 2 last access: 5.5 min ago → removed
```

---

## Request Flow Example

### Scenario: Client makes 105 requests to /api/upload

**Config**: Capacity 10, Refill 1/sec

```
Request 1:  10 tokens → Allow (9 remaining)
Request 2:  9 tokens  → Allow (8 remaining)
...
Request 10: 1 token   → Allow (0 remaining)
Request 11: 0 tokens  → DENY (429 Too Many Requests)

[wait 1 second]

Request 12: 1 token   → Allow (refilled 1 token)
Request 13: 0 tokens  → DENY

[wait 10 seconds]

Request 14: 10 tokens → Allow (refilled 10 tokens)
```

---

## Testing the Solution

### Test 1: Normal Usage

```bash
# First request
curl -i http://localhost:8080/api/posts
```

**Response**:
```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 99
X-RateLimit-Reset: 1642012920
Content-Type: application/json

[]
```

### Test 2: Exhaust Tokens

```bash
# Send 110 requests quickly
for i in {1..110}; do
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/api/posts
done
```

**Expected**:
```
200
200
... (100 times)
429
429
... (10 times)
```

### Test 3: Token Refilling

```bash
# Use all tokens
for i in {1..100}; do
  curl -s http://localhost:8080/api/posts > /dev/null
done

# Check - should be rate limited
curl -i http://localhost:8080/api/posts
# 429 Too Many Requests

# Wait 10 seconds (100 tokens refill at 10/sec)
sleep 10

# Check - should work again
curl -i http://localhost:8080/api/posts
# 200 OK, X-RateLimit-Remaining: 99
```

### Test 4: Endpoint-Specific Limits

```bash
# Upload has lower limit (10 capacity)
for i in {1..15}; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:8080/api/upload)
  echo "Upload $i: $STATUS"
done
```

**Expected**:
```
Upload 1: 200
Upload 2: 200
...
Upload 10: 200
Upload 11: 429
Upload 12: 429
...
```

---

## Performance Characteristics

### Memory Usage

```go
// Per client-endpoint pair:
- TokenBucket: ~80 bytes
- lastAccess: ~24 bytes
- map overhead: ~48 bytes
Total: ~152 bytes per active client-endpoint

1000 active clients * 3 endpoints = ~456 KB
10000 active clients * 3 endpoints = ~4.5 MB
```

**With cleanup**: Old entries removed, memory stays bounded

### CPU Usage

```go
// Per request:
- 1 map lookup (read lock)
- Time calculation (~10 ns)
- Token arithmetic (~5 ns)
- 1 map update (write lock, if new)

Total: ~1-5 microseconds per request
```

**Very low overhead** - suitable for high-traffic APIs

---

## Common Issues and Solutions

### Issue 1: Bursts at Bucket Refill

**Problem**: Client can burst 100 requests, wait 10 seconds, burst 100 again.

**Solution**: Adjust capacity and refill rate
```go
// Instead of: 100 capacity, 10/sec refill
// Use: 60 capacity, 10/sec refill
// Allows 60 burst, then sustained 10/sec
```

### Issue 2: Shared IPs (NAT/Proxy)

**Problem**: Multiple users behind same IP share limit.

**Solution**: Use authenticated user ID
```go
func (rl *RateLimiter) getClientID(r *http.Request) string {
    // Try to get user ID from JWT
    if userID := getUserFromJWT(r); userID != "" {
        return "user:" + userID
    }
    
    // Fallback to IP
    return getIPFromRequest(r)
}
```

### Issue 3: Clock Skew

**Problem**: System clock changes can affect refilling.

**Solution**: Use monotonic time
```go
// Go's time.Now() uses monotonic clock for duration calculations
elapsed := now.Sub(tb.lastRefill)  // Uses monotonic clock
```

---

## What You've Learned

✅ **Token bucket algorithm** with continuous refilling  
✅ **Thread-safe operations** with mutex  
✅ **Middleware pattern** for rate limiting  
✅ **Per-client, per-endpoint** limits  
✅ **Rate limit headers** (standard format)  
✅ **Cleanup goroutine** to prevent memory leaks  
✅ **Reset time calculation** for client feedback  
✅ **Endpoint-specific configuration**  

You now understand in-memory rate limiting suitable for single-server APIs!
