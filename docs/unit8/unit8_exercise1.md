# Unit 8 - Exercise 1: Token Bucket Rate Limiter

**Difficulty**: Intermediate  
**Estimated Time**: 45-60 minutes  
**Concepts Covered**: Token bucket algorithm, middleware, rate limit headers, cleanup

---

## Objective

Implement a token bucket rate limiter for an API that:
- Limits requests per client (by IP address)
- Uses token bucket algorithm for smooth rate limiting
- Returns proper HTTP 429 responses
- Includes rate limit headers
- Cleans up inactive limiters
- Supports different limits per endpoint

---

## Requirements

### Rate Limit Configuration

| Endpoint | Limit | Refill Rate |
|----------|-------|-------------|
| /api/posts | 100 capacity | 10 tokens/second |
| /api/upload | 10 capacity | 1 token/second |
| /api/search | 50 capacity | 5 tokens/second |
| Default | 100 capacity | 10 tokens/second |

### Response Headers

```
X-RateLimit-Limit: 100           # Bucket capacity
X-RateLimit-Remaining: 45        # Tokens remaining
X-RateLimit-Reset: 1642012800    # Unix timestamp when full
Retry-After: 10                  # Seconds to wait (when limited)
```

### HTTP Status

- **200 OK**: Request allowed
- **429 Too Many Requests**: Rate limit exceeded

---

## Starter Code

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
    refillRate float64   // tokens per second
    lastRefill time.Time
    mu         sync.Mutex
}

// TODO: Implement NewTokenBucket
func NewTokenBucket(capacity, refillRate float64) *TokenBucket {
    // Create bucket with full capacity
    // Set lastRefill to now
}

// TODO: Implement Allow
func (tb *TokenBucket) Allow() bool {
    // 1. Lock mutex
    // 2. Calculate elapsed time since last refill
    // 3. Add tokens based on refill rate
    // 4. Cap tokens at capacity
    // 5. Update lastRefill
    // 6. Check if >= 1 token available
    // 7. If yes: consume 1 token and return true
    // 8. If no: return false
}

// TODO: Implement Remaining
func (tb *TokenBucket) Remaining() int {
    // Return current token count (integer)
}

// TODO: Implement ResetTime
func (tb *TokenBucket) ResetTime() time.Time {
    // Calculate when bucket will be full again
    // based on current tokens and refill rate
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
}

type LimitConfig struct {
    Capacity   float64
    RefillRate float64
}

// TODO: Implement NewRateLimiter
func NewRateLimiter(defaultConfig LimitConfig) *RateLimiter {
    // Create rate limiter
    // Start cleanup goroutine
    // Initialize endpoint limits
}

// TODO: Implement GetLimiter
func (rl *RateLimiter) GetLimiter(clientID string, config LimitConfig) *TokenBucket {
    // 1. Check if limiter exists (read lock)
    // 2. If exists, return it
    // 3. If not, create new limiter (write lock)
    // 4. Store and return
}

// TODO: Implement getClientID
func (rl *RateLimiter) getClientID(r *http.Request) string {
    // Extract IP from RemoteAddr
    // Remove port number
    // Return clean IP
}

// TODO: Implement getLimitConfig
func (rl *RateLimiter) getLimitConfig(endpoint string) LimitConfig {
    // Check if endpoint has specific config
    // Return endpoint config or default
}

// TODO: Implement Middleware
func (rl *RateLimiter) Middleware(next http.Handler) http.Handler {
    // 1. Get client ID
    // 2. Get limit config for endpoint
    // 3. Get or create limiter
    // 4. Check if request allowed
    // 5. If not allowed:
    //    - Set rate limit headers (limit, remaining=0, retry-after)
    //    - Return 429
    // 6. If allowed:
    //    - Set rate limit headers (limit, remaining, reset)
    //    - Call next handler
}

// TODO: Implement cleanup
func (rl *RateLimiter) cleanup() {
    // Remove limiters that haven't been used in 5 minutes
    // Helps prevent memory leaks
}

// TODO: Implement startCleanup
func (rl *RateLimiter) startCleanup() {
    // Start goroutine with ticker
    // Run cleanup every 1 minute
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
    posts    = []Post{}
    postsMu  sync.RWMutex
    nextID   = 1
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
    // TODO: Create rate limiter with default config
    // rateLimiter := NewRateLimiter(LimitConfig{
    //     Capacity:   100,
    //     RefillRate: 10,
    // })
    
    // TODO: Set endpoint-specific limits
    // rateLimiter.endpointLimits = map[string]LimitConfig{
    //     "/api/upload": {Capacity: 10, RefillRate: 1},
    //     "/api/search": {Capacity: 50, RefillRate: 5},
    // }
    
    r := mux.NewRouter()
    
    // TODO: Apply rate limiter middleware
    
    // Routes
    r.HandleFunc("/api/posts", getPosts).Methods("GET")
    r.HandleFunc("/api/posts", createPost).Methods("POST")
    r.HandleFunc("/api/upload", uploadFile).Methods("POST")
    r.HandleFunc("/api/search", searchPosts).Methods("GET")
    
    fmt.Println("Server starting on :8080")
    fmt.Println("Rate limits:")
    fmt.Println("  /api/posts  - 100 requests, 10/sec refill")
    fmt.Println("  /api/upload - 10 requests, 1/sec refill")
    fmt.Println("  /api/search - 50 requests, 5/sec refill")
    
    http.ListenAndServe(":8080", r)
}
```

---

## Your Tasks

### Task 1: Implement TokenBucket

Create token bucket with:
- `NewTokenBucket(capacity, refillRate)`: Initialize bucket
- `Allow()`: Check and consume token
- `Remaining()`: Get current token count
- `ResetTime()`: Calculate when bucket will be full

### Task 2: Implement Token Refilling

In `Allow()`:
- Calculate elapsed time since last refill
- Add tokens: `tokens += elapsed * refillRate`
- Cap at capacity
- Update lastRefill timestamp

### Task 3: Implement RateLimiter

Create rate limiter with:
- Map of client ID → TokenBucket
- Thread-safe access (RWMutex)
- Cleanup goroutine
- Endpoint-specific limits

### Task 4: Implement Middleware

Rate limiter middleware should:
1. Extract client ID (IP address)
2. Get limit config for endpoint
3. Get or create token bucket
4. Check if request allowed
5. Set appropriate headers
6. Return 429 or allow request

### Task 5: Implement Cleanup

Background cleanup:
- Remove limiters inactive for 5+ minutes
- Run every 1 minute
- Prevent memory leaks

### Task 6: Set Rate Limit Headers

Add headers to response:
```go
X-RateLimit-Limit: {capacity}
X-RateLimit-Remaining: {tokens}
X-RateLimit-Reset: {resetTime.Unix()}
Retry-After: {seconds}  // Only when rate limited
```

---

## Testing Your Implementation

### Test 1: Normal Usage

```bash
# Should succeed (first requests)
for i in {1..10}; do
  curl -i http://localhost:8080/api/posts
done

# Check headers
curl -i http://localhost:8080/api/posts | grep X-RateLimit
```

### Test 2: Hit Rate Limit

```bash
# Burst 150 requests (capacity is 100)
for i in {1..150}; do
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/api/posts
done

# First 100 should return 200
# Next 50 should return 429
```

### Test 3: Token Refilling

```bash
# Use all tokens
for i in {1..100}; do
  curl -s http://localhost:8080/api/posts > /dev/null
done

# Wait 10 seconds (100 tokens refill at 10/sec)
sleep 10

# Should succeed again
curl -i http://localhost:8080/api/posts
```

### Test 4: Endpoint-Specific Limits

```bash
# Upload has lower limit (10 capacity, 1/sec refill)
for i in {1..15}; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:8080/api/upload)
  echo "Request $i: $STATUS"
done

# First 10 should be 200
# Next 5 should be 429
```

### Test 5: Different Clients

```bash
# Client 1 (via proxy)
for i in {1..100}; do
  curl -s http://localhost:8080/api/posts > /dev/null
done

# Client 2 (different IP) should have separate limit
# In practice, test from different machines or use X-Forwarded-For
```

---

## Expected Behavior

### First Request
```
GET /api/posts
Response: 200 OK
Headers:
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 99
  X-RateLimit-Reset: 1642012910
```

### After 100 Requests
```
GET /api/posts
Response: 429 Too Many Requests
Headers:
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 0
  X-RateLimit-Reset: 1642012920
  Retry-After: 10
Body:
  {"error": "Rate limit exceeded"}
```

### After Waiting 10 Seconds
```
GET /api/posts
Response: 200 OK
Headers:
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 99
```

---

## Bonus Challenges

### Bonus 1: Per-User Rate Limiting

Use authenticated user ID instead of IP:
```go
func (rl *RateLimiter) getClientID(r *http.Request) string {
    // Try to get user ID from context/JWT
    userID := r.Context().Value("user_id")
    if userID != nil {
        return fmt.Sprintf("user:%s", userID)
    }
    
    // Fallback to IP
    return getIPFromRequest(r)
}
```

### Bonus 2: Burst Capacity

Allow short bursts above normal rate:
```go
type BurstTokenBucket struct {
    *TokenBucket
    burstCapacity float64
}
```

### Bonus 3: Cost-Based Rate Limiting

Different endpoints consume different token amounts:
```go
func (tb *TokenBucket) AllowCost(cost float64) bool {
    if tb.tokens < cost {
        return false
    }
    tb.tokens -= cost
    return true
}

// Upload costs 10 tokens, search costs 5, etc.
```

### Bonus 4: Rate Limit Metrics

Track rate limit hits:
```go
type Metrics struct {
    TotalRequests   int64
    LimitedRequests int64
    mu              sync.Mutex
}

func (m *Metrics) RecordRequest(limited bool) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.TotalRequests++
    if limited {
        m.LimitedRequests++
    }
}
```

### Bonus 5: Whitelist/Blacklist

Skip rate limiting for certain IPs:
```go
func (rl *RateLimiter) isWhitelisted(ip string) bool {
    whitelist := []string{"127.0.0.1", "192.168.1.100"}
    for _, allowed := range whitelist {
        if ip == allowed {
            return true
        }
    }
    return false
}
```

---

## Hints

### Hint 1: Token Refilling

```go
func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()
    
    now := time.Now()
    elapsed := now.Sub(tb.lastRefill).Seconds()
    
    // Add tokens
    tb.tokens += elapsed * tb.refillRate
    
    // Cap at capacity
    if tb.tokens > tb.capacity {
        tb.tokens = tb.capacity
    }
    
    tb.lastRefill = now
    
    // Check availability
    if tb.tokens < 1 {
        return false
    }
    
    tb.tokens--
    return true
}
```

### Hint 2: Reset Time Calculation

```go
func (tb *TokenBucket) ResetTime() time.Time {
    tb.mu.Lock()
    defer tb.mu.Unlock()
    
    if tb.tokens >= tb.capacity {
        return time.Now()
    }
    
    tokensNeeded := tb.capacity - tb.tokens
    secondsToFill := tokensNeeded / tb.refillRate
    
    return time.Now().Add(time.Duration(secondsToFill) * time.Second)
}
```

### Hint 3: Client ID Extraction

```go
func (rl *RateLimiter) getClientID(r *http.Request) string {
    // Get IP from RemoteAddr
    ip := r.RemoteAddr
    
    // Remove port
    if idx := strings.LastIndex(ip, ":"); idx != -1 {
        ip = ip[:idx]
    }
    
    return ip
}
```

---

## What You're Learning

✅ **Token bucket algorithm** for rate limiting  
✅ **Middleware pattern** for cross-cutting concerns  
✅ **Thread-safe operations** with mutex  
✅ **Rate limit headers** (standard format)  
✅ **HTTP 429** status code  
✅ **Resource cleanup** to prevent memory leaks  
✅ **Endpoint-specific limits**  
✅ **Token refilling** calculation  

This is the foundation for production-grade rate limiting!
