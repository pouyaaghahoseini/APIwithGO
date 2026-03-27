# Unit 6 - Exercise 2: Redis Cache for Blog API

**Difficulty**: Advanced  
**Estimated Time**: 60-75 minutes  
**Concepts Covered**: Redis caching, cache stampede prevention, write-through pattern, cache warming, distributed caching

---

## Objective

Implement Redis caching for a blog API with advanced patterns:
- Cache stampede prevention using mutex
- Write-through caching for updates
- Cache warming on startup
- Distributed cache invalidation
- Cache metrics and monitoring

---

## Requirements

### API Endpoints

| Method | Path | Description | Cache Strategy |
|--------|------|-------------|----------------|
| GET | /posts | List posts | Cache 5 min, warm on startup |
| GET | /posts/{id} | Get post | Cache 10 min, stampede prevention |
| POST | /posts | Create post | Write-through |
| PUT | /posts/{id} | Update post | Write-through + invalidate |
| DELETE | /posts/{id} | Delete post | Invalidate related caches |
| GET | /posts/{id}/comments | Get comments | Cache 3 min |
| POST | /posts/{id}/comments | Add comment | Invalidate post cache |
| GET | /stats/cache | Cache statistics | No cache |
| POST | /cache/warm | Trigger cache warming | Admin only |
| DELETE | /cache/clear | Clear all cache | Admin only |

### Models

```go
type Post struct {
    ID           int       `json:"id"`
    Title        string    `json:"title"`
    Content      string    `json:"content"`
    Author       string    `json:"author"`
    ViewCount    int       `json:"view_count"`
    CommentCount int       `json:"comment_count"`
    Tags         []string  `json:"tags"`
    CreatedAt    time.Time `json:"created_at"`
    UpdatedAt    time.Time `json:"updated_at"`
}

type Comment struct {
    ID        int       `json:"id"`
    PostID    int       `json:"post_id"`
    Author    string    `json:"author"`
    Content   string    `json:"content"`
    CreatedAt time.Time `json:"created_at"`
}

type CacheMetrics struct {
    Hits         int64         `json:"hits"`
    Misses       int64         `json:"misses"`
    Size         int64         `json:"size"`
    HitRate      float64       `json:"hit_rate"`
    AvgLatency   time.Duration `json:"avg_latency_ms"`
    StampedePrevented int64    `json:"stampede_prevented"`
}
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
    "sync"
    "sync/atomic"
    "time"

    "github.com/gorilla/mux"
    "github.com/redis/go-redis/v9"
)

var ctx = context.Background()

type Post struct {
    ID           int       `json:"id"`
    Title        string    `json:"title"`
    Content      string    `json:"content"`
    Author       string    `json:"author"`
    ViewCount    int       `json:"view_count"`
    CommentCount int       `json:"comment_count"`
    Tags         []string  `json:"tags"`
    CreatedAt    time.Time `json:"created_at"`
    UpdatedAt    time.Time `json:"updated_at"`
}

type Comment struct {
    ID        int       `json:"id"`
    PostID    int       `json:"post_id"`
    Author    string    `json:"author"`
    Content   string    `json:"content"`
    CreatedAt time.Time `json:"created_at"`
}

type CacheMetrics struct {
    Hits              int64         `json:"hits"`
    Misses            int64         `json:"misses"`
    Size              int64         `json:"size"`
    HitRate           float64       `json:"hit_rate"`
    AvgLatencyMs      float64       `json:"avg_latency_ms"`
    StampedePrevented int64         `json:"stampede_prevented"`
}

type RedisCache struct {
    client            *redis.Client
    hits              int64
    misses            int64
    stampedePrevented int64
    totalLatency      int64
    requests          int64
    mu                sync.Map // For stampede prevention
}

// Storage
var (
    posts      = make(map[int]Post)
    comments   = make(map[int][]Comment)
    nextPostID = 1
    nextCommID = 1
    storageMu  sync.RWMutex
    cache      *RedisCache
)

// TODO: Implement NewRedisCache
func NewRedisCache(addr string) *RedisCache {
    // Hint: Create Redis client, test connection, initialize metrics
}

// TODO: Implement Set
func (c *RedisCache) Set(key string, value interface{}, ttl time.Duration) error {
    // Hint: Marshal to JSON, store in Redis with expiration
}

// TODO: Implement Get with stampede prevention
func (c *RedisCache) Get(key string, dest interface{}) error {
    // Hint: 
    // 1. Try to get from Redis
    // 2. Update metrics
    // 3. Track latency
}

// TODO: Implement GetOrLoad with mutex for stampede prevention
func (c *RedisCache) GetOrLoad(key string, dest interface{}, ttl time.Duration, 
    loader func() (interface{}, error)) error {
    // Hint:
    // 1. Try cache first
    // 2. If miss, acquire mutex for this key
    // 3. Double-check cache (another goroutine might have loaded it)
    // 4. If still miss, call loader function
    // 5. Cache result and return
    // 6. Track stampede prevention
}

// TODO: Implement Delete
func (c *RedisCache) Delete(key string) error {
    // Hint: Delete from Redis
}

// TODO: Implement DeletePattern
func (c *RedisCache) DeletePattern(pattern string) error {
    // Hint: Use SCAN to find matching keys, then delete
}

// TODO: Implement Clear
func (c *RedisCache) Clear() error {
    // Hint: Use FLUSHDB
}

// TODO: Implement GetMetrics
func (c *RedisCache) GetMetrics() CacheMetrics {
    // Hint: Calculate hit rate, average latency, get size from Redis
}

// TODO: Implement getPosts with cache warming
func getPosts(w http.ResponseWriter, r *http.Request) {
    cacheKey := "posts:all"
    
    // Try cache with stampede prevention
    // On miss: query database
}

// TODO: Implement getPost with stampede prevention
func getPost(w http.ResponseWriter, r *http.Request) {
    // Extract ID
    // Use GetOrLoad to prevent cache stampede
    // Increment view count
}

// TODO: Implement createPost with write-through
func createPost(w http.ResponseWriter, r *http.Request) {
    // Create post in database
    // Cache immediately (write-through)
    // Invalidate list cache
}

// TODO: Implement updatePost with write-through
func updatePost(w http.ResponseWriter, r *http.Request) {
    // Update database
    // Update cache immediately
    // Invalidate list cache
}

// TODO: Implement deletePost
func deletePost(w http.ResponseWriter, r *http.Request) {
    // Delete from database
    // Invalidate all related caches
}

// TODO: Implement getComments
func getComments(w http.ResponseWriter, r *http.Request) {
    // Cache comments for 3 minutes
}

// TODO: Implement createComment
func createComment(w http.ResponseWriter, r *http.Request) {
    // Add comment
    // Invalidate comment cache
    // Invalidate post cache (comment count changed)
}

// TODO: Implement getCacheStats
func getCacheStats(w http.ResponseWriter, r *http.Request) {
    // Return cache metrics
}

// TODO: Implement warmCache
func warmCache(w http.ResponseWriter, r *http.Request) {
    // Load top posts into cache
    // Load recent comments
    // Return warming statistics
}

// TODO: Implement clearCache
func clearCache(w http.ResponseWriter, r *http.Request) {
    // Clear all cache
    // Return success message
}

func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}

func seedDatabase() {
    storageMu.Lock()
    defer storageMu.Unlock()

    now := time.Now()

    posts[1] = Post{
        ID:           1,
        Title:        "Introduction to Go",
        Content:      "Go is a great language...",
        Author:       "john",
        ViewCount:    0,
        CommentCount: 0,
        Tags:         []string{"golang", "tutorial"},
        CreatedAt:    now,
        UpdatedAt:    now,
    }

    posts[2] = Post{
        ID:           2,
        Title:        "Redis Caching",
        Content:      "Redis is a fast key-value store...",
        Author:       "jane",
        ViewCount:    0,
        CommentCount: 0,
        Tags:         []string{"redis", "cache"},
        CreatedAt:    now,
        UpdatedAt:    now,
    }

    nextPostID = 3
}

func main() {
    seedDatabase()

    // TODO: Initialize Redis cache
    cache = NewRedisCache("localhost:6379")

    // TODO: Warm cache on startup
    // warmCacheOnStartup()

    r := mux.NewRouter()

    // TODO: Register routes

    fmt.Println("Server starting on :8080")
    fmt.Println("Make sure Redis is running: docker run -d -p 6379:6379 redis:latest")
    http.ListenAndServe(":8080", r)
}
```

---

## Your Tasks

### Task 1: Implement RedisCache

Create Redis cache wrapper with:
- Connection management
- Metrics tracking (hits, misses, latency)
- Stampede prevention using `sync.Map` of mutexes
- Error handling

### Task 2: Implement Cache Methods

- `Set(key, value, ttl)`: Store with expiration
- `Get(key, dest)`: Retrieve with metrics
- `GetOrLoad(key, dest, ttl, loader)`: Stampede prevention
- `Delete(key)`: Remove single key
- `DeletePattern(pattern)`: Remove matching keys
- `Clear()`: Clear all cache
- `GetMetrics()`: Return statistics

### Task 3: Cache Stampede Prevention

Implement `GetOrLoad` with mutex:
```go
// When cache misses on popular item:
// 1. First goroutine acquires mutex for that key
// 2. Other goroutines wait on same mutex
// 3. First loads data and caches it
// 4. Others get cached result
// 5. Track how many goroutines were prevented from hitting DB
```

### Task 4: Write-Through Pattern

For POST/PUT operations:
```go
// 1. Write to database
// 2. Write to cache immediately
// 3. Return response
```

### Task 5: Cache Warming

Implement cache warming:
- Load top 10 posts on startup
- Load recent comments
- Return statistics about what was warmed

### Task 6: Admin Endpoints

- `/stats/cache`: Show metrics
- `/cache/warm`: Trigger manual warming
- `/cache/clear`: Clear all cache

---

## Testing Your Implementation

### Test Cache Stampede Prevention

```bash
# Simulate high traffic to same post
# Run in parallel (10 concurrent requests)
seq 10 | xargs -n1 -P10 curl http://localhost:8080/posts/1

# Check stats - stampede_prevented should be > 0
curl http://localhost:8080/stats/cache
```

### Test Write-Through

```bash
# Create post
curl -X POST http://localhost:8080/posts \
  -H "Content-Type: application/json" \
  -d '{"title":"New Post","content":"Content here"}'

# Immediately read - should be cached (X-Cache: HIT)
curl -i http://localhost:8080/posts/3
```

### Test Cache Invalidation

```bash
# Cache a post
curl http://localhost:8080/posts/1

# Add comment (invalidates post cache)
curl -X POST http://localhost:8080/posts/1/comments \
  -H "Content-Type: application/json" \
  -d '{"author":"bob","content":"Great post!"}'

# Read post again - should be MISS (cache invalidated)
curl -i http://localhost:8080/posts/1
```

### Test Cache Warming

```bash
# Clear cache
curl -X DELETE http://localhost:8080/cache/clear

# Trigger warming
curl -X POST http://localhost:8080/cache/warm

# Subsequent requests should be HITs
curl -i http://localhost:8080/posts/1
```

### Test Redis Persistence

```bash
# Cache some data
curl http://localhost:8080/posts/1

# Restart Go app (Redis keeps running)

# Data still cached
curl -i http://localhost:8080/posts/1
# Should see: X-Cache: HIT
```

---

## Expected Performance

### Without Cache
```
GET /posts/1: 50-100ms (database query)
10 concurrent requests: 50-100ms each = 500-1000ms total DB time
```

### With Cache (No Stampede Prevention)
```
First request: 50-100ms (MISS)
Next 9 requests during load: 50-100ms each (all hit DB)
Total: 500-1000ms DB time
```

### With Cache + Stampede Prevention
```
First request: 50-100ms (MISS, loads from DB)
Next 9 requests: Wait for first, then <1ms (from cache)
Total: ~50-100ms DB time
Stampede prevented: 9 goroutines
```

---

## Bonus Challenges

### Bonus 1: Cache Tiering
Implement two-level cache:
```go
// L1: In-memory (fast, small)
// L2: Redis (shared, larger)
// Check L1 first, then L2, then DB
```

### Bonus 2: Stale-While-Revalidate
```go
// If cached but near expiration:
// - Return cached data immediately
// - Refresh cache in background
```

### Bonus 3: Cache Tags
```go
// Tag posts by category
cache.Set("post:1", post, 10*time.Minute, Tags("posts", "golang"))

// Invalidate by tag
cache.InvalidateByTag("golang")
```

### Bonus 4: Compression
```go
// Compress large payloads before caching
// Saves Redis memory
```

### Bonus 5: Cache Preloading
```go
// Webhook endpoint: POST /cache/preload
// Accepts list of post IDs to preload
```

---

## Hints

### Hint 1: Stampede Prevention with Mutex

```go
func (c *RedisCache) GetOrLoad(key string, dest interface{}, ttl time.Duration,
    loader func() (interface{}, error)) error {
    
    // Try cache first
    err := c.Get(key, dest)
    if err == nil {
        return nil // Cache hit
    }
    
    // Get or create mutex for this key
    muInterface, _ := c.mu.LoadOrStore(key, &sync.Mutex{})
    mu := muInterface.(*sync.Mutex)
    
    mu.Lock()
    defer mu.Unlock()
    
    // Double-check cache (another goroutine might have loaded it)
    err = c.Get(key, dest)
    if err == nil {
        atomic.AddInt64(&c.stampedePrevented, 1)
        return nil
    }
    
    // Load from source
    data, err := loader()
    if err != nil {
        return err
    }
    
    // Cache it
    c.Set(key, data, ttl)
    
    // Copy to dest
    bytes, _ := json.Marshal(data)
    json.Unmarshal(bytes, dest)
    
    return nil
}
```

### Hint 2: Pattern Deletion with SCAN

```go
func (c *RedisCache) DeletePattern(pattern string) error {
    iter := c.client.Scan(ctx, 0, pattern, 0).Iterator()
    
    keys := []string{}
    for iter.Next(ctx) {
        keys = append(keys, iter.Val())
    }
    
    if len(keys) > 0 {
        return c.client.Del(ctx, keys...).Err()
    }
    
    return nil
}
```

### Hint 3: Cache Warming

```go
func warmCacheOnStartup() {
    log.Println("Warming cache...")
    
    storageMu.RLock()
    defer storageMu.RUnlock()
    
    // Cache all posts
    for id, post := range posts {
        cacheKey := fmt.Sprintf("post:%d", id)
        cache.Set(cacheKey, post, 10*time.Minute)
    }
    
    // Cache post list
    postList := []Post{}
    for _, post := range posts {
        postList = append(postList, post)
    }
    cache.Set("posts:all", postList, 5*time.Minute)
    
    log.Printf("Cache warmed: %d items", len(posts))
}
```

---

## What You're Learning

✅ **Redis integration** with go-redis  
✅ **Cache stampede prevention** with mutexes  
✅ **Write-through caching** pattern  
✅ **Cache warming** strategies  
✅ **Distributed caching** across servers  
✅ **Pattern-based invalidation** with SCAN  
✅ **Cache metrics** and monitoring  
✅ **Production caching patterns**  

This exercise demonstrates enterprise-level caching with Redis!
