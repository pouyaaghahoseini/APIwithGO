# Unit 6 - Exercise 2 Solution: Redis Cache for Blog API

**Complete implementation with advanced caching patterns**

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
    "sync"
    "sync/atomic"
    "time"

    "github.com/gorilla/mux"
    "github.com/redis/go-redis/v9"
)

var ctx = context.Background()

// =============================================================================
// MODELS
// =============================================================================

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
    Hits              int64   `json:"hits"`
    Misses            int64   `json:"misses"`
    Size              int64   `json:"size"`
    HitRate           float64 `json:"hit_rate"`
    AvgLatencyMs      float64 `json:"avg_latency_ms"`
    StampedePrevented int64   `json:"stampede_prevented"`
}

// =============================================================================
// REDIS CACHE
// =============================================================================

type RedisCache struct {
    client            *redis.Client
    hits              int64
    misses            int64
    stampedePrevented int64
    totalLatency      int64
    requests          int64
    mu                sync.Map // Map of key -> *sync.Mutex for stampede prevention
}

func NewRedisCache(addr string) *RedisCache {
    client := redis.NewClient(&redis.Options{
        Addr:     addr,
        Password: "",
        DB:       0,
    })

    // Test connection
    _, err := client.Ping(ctx).Result()
    if err != nil {
        panic(fmt.Sprintf("Failed to connect to Redis: %v", err))
    }

    return &RedisCache{
        client: client,
    }
}

func (c *RedisCache) Set(key string, value interface{}, ttl time.Duration) error {
    data, err := json.Marshal(value)
    if err != nil {
        return err
    }

    return c.client.Set(ctx, key, data, ttl).Err()
}

func (c *RedisCache) Get(key string, dest interface{}) error {
    start := time.Now()

    val, err := c.client.Get(ctx, key).Result()

    // Track latency
    latency := time.Since(start).Microseconds()
    atomic.AddInt64(&c.totalLatency, latency)
    atomic.AddInt64(&c.requests, 1)

    if err == redis.Nil {
        atomic.AddInt64(&c.misses, 1)
        return fmt.Errorf("key not found")
    } else if err != nil {
        atomic.AddInt64(&c.misses, 1)
        return err
    }

    atomic.AddInt64(&c.hits, 1)
    return json.Unmarshal([]byte(val), dest)
}

// GetOrLoad prevents cache stampede using mutex per key
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

func (c *RedisCache) Delete(key string) error {
    return c.client.Del(ctx, key).Err()
}

func (c *RedisCache) DeletePattern(pattern string) error {
    // Use SCAN to find matching keys
    iter := c.client.Scan(ctx, 0, pattern, 0).Iterator()

    keys := []string{}
    for iter.Next(ctx) {
        keys = append(keys, iter.Val())
    }

    if err := iter.Err(); err != nil {
        return err
    }

    // Delete all matching keys
    if len(keys) > 0 {
        return c.client.Del(ctx, keys...).Err()
    }

    return nil
}

func (c *RedisCache) Clear() error {
    return c.client.FlushDB(ctx).Err()
}

func (c *RedisCache) GetMetrics() CacheMetrics {
    hits := atomic.LoadInt64(&c.hits)
    misses := atomic.LoadInt64(&c.misses)
    requests := atomic.LoadInt64(&c.requests)
    totalLatency := atomic.LoadInt64(&c.totalLatency)

    // Calculate hit rate
    total := hits + misses
    hitRate := 0.0
    if total > 0 {
        hitRate = float64(hits) / float64(total) * 100
    }

    // Calculate average latency
    avgLatency := 0.0
    if requests > 0 {
        avgLatency = float64(totalLatency) / float64(requests) / 1000.0 // Convert to ms
    }

    // Get cache size from Redis
    size, _ := c.client.DBSize(ctx).Result()

    return CacheMetrics{
        Hits:              hits,
        Misses:            misses,
        Size:              size,
        HitRate:           hitRate,
        AvgLatencyMs:      avgLatency,
        StampedePrevented: atomic.LoadInt64(&c.stampedePrevented),
    }
}

// =============================================================================
// STORAGE
// =============================================================================

var (
    posts      = make(map[int]Post)
    comments   = make(map[int][]Comment)
    nextPostID = 1
    nextCommID = 1
    storageMu  sync.RWMutex
    cache      *RedisCache
)

// =============================================================================
// HANDLERS
// =============================================================================

func getPosts(w http.ResponseWriter, r *http.Request) {
    cacheKey := "posts:all"

    var postList []Post

    // Use GetOrLoad with stampede prevention
    err := cache.GetOrLoad(cacheKey, &postList, 5*time.Minute, func() (interface{}, error) {
        storageMu.RLock()
        defer storageMu.RUnlock()

        list := make([]Post, 0, len(posts))
        for _, post := range posts {
            list = append(list, post)
        }
        return list, nil
    })

    if err != nil {
        respondError(w, http.StatusInternalServerError, "Failed to load posts")
        return
    }

    w.Header().Set("X-Cache", "CACHED")
    respondJSON(w, http.StatusOK, postList)
}

func getPost(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    cacheKey := fmt.Sprintf("post:%d", id)

    var post Post

    // Use GetOrLoad to prevent cache stampede
    err := cache.GetOrLoad(cacheKey, &post, 10*time.Minute, func() (interface{}, error) {
        storageMu.RLock()
        defer storageMu.RUnlock()

        p, exists := posts[id]
        if !exists {
            return nil, fmt.Errorf("post not found")
        }
        return p, nil
    })

    if err != nil {
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    // Increment view count
    storageMu.Lock()
    p := posts[id]
    p.ViewCount++
    posts[id] = p
    storageMu.Unlock()

    // Update cache with new view count (write-through)
    cache.Set(cacheKey, p, 10*time.Minute)

    respondJSON(w, http.StatusOK, post)
}

func createPost(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Title   string   `json:"title"`
        Content string   `json:"content"`
        Author  string   `json:"author"`
        Tags    []string `json:"tags"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Validate
    if req.Title == "" || req.Content == "" {
        respondError(w, http.StatusBadRequest, "Title and content are required")
        return
    }

    // Create post
    now := time.Now()
    storageMu.Lock()
    post := Post{
        ID:           nextPostID,
        Title:        req.Title,
        Content:      req.Content,
        Author:       req.Author,
        ViewCount:    0,
        CommentCount: 0,
        Tags:         req.Tags,
        CreatedAt:    now,
        UpdatedAt:    now,
    }
    posts[nextPostID] = post
    nextPostID++
    storageMu.Unlock()

    // Write-through: cache immediately
    cacheKey := fmt.Sprintf("post:%d", post.ID)
    cache.Set(cacheKey, post, 10*time.Minute)

    // Invalidate list cache
    cache.Delete("posts:all")

    respondJSON(w, http.StatusCreated, post)
}

func updatePost(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    var req struct {
        Title   string   `json:"title"`
        Content string   `json:"content"`
        Tags    []string `json:"tags"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Update post
    storageMu.Lock()
    post, exists := posts[id]
    if !exists {
        storageMu.Unlock()
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    post.Title = req.Title
    post.Content = req.Content
    post.Tags = req.Tags
    post.UpdatedAt = time.Now()
    posts[id] = post
    storageMu.Unlock()

    // Write-through: update cache immediately
    cacheKey := fmt.Sprintf("post:%d", id)
    cache.Set(cacheKey, post, 10*time.Minute)

    // Invalidate list cache
    cache.Delete("posts:all")

    respondJSON(w, http.StatusOK, post)
}

func deletePost(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    storageMu.Lock()
    _, exists := posts[id]
    if exists {
        delete(posts, id)
        delete(comments, id)
    }
    storageMu.Unlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    // Invalidate all related caches
    cache.Delete(fmt.Sprintf("post:%d", id))
    cache.Delete(fmt.Sprintf("comments:%d", id))
    cache.Delete("posts:all")

    w.WriteHeader(http.StatusNoContent)
}

func getComments(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    postID, _ := strconv.Atoi(vars["id"])

    cacheKey := fmt.Sprintf("comments:%d", postID)

    var commentList []Comment

    err := cache.GetOrLoad(cacheKey, &commentList, 3*time.Minute, func() (interface{}, error) {
        storageMu.RLock()
        defer storageMu.RUnlock()

        list, exists := comments[postID]
        if !exists {
            return []Comment{}, nil
        }
        return list, nil
    })

    if err != nil {
        respondError(w, http.StatusInternalServerError, "Failed to load comments")
        return
    }

    respondJSON(w, http.StatusOK, commentList)
}

func createComment(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    postID, _ := strconv.Atoi(vars["id"])

    var req struct {
        Author  string `json:"author"`
        Content string `json:"content"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Validate
    if req.Content == "" {
        respondError(w, http.StatusBadRequest, "Content is required")
        return
    }

    // Create comment
    storageMu.Lock()
    comment := Comment{
        ID:        nextCommID,
        PostID:    postID,
        Author:    req.Author,
        Content:   req.Content,
        CreatedAt: time.Now(),
    }
    comments[postID] = append(comments[postID], comment)
    nextCommID++

    // Update comment count on post
    post := posts[postID]
    post.CommentCount++
    posts[postID] = post
    storageMu.Unlock()

    // Invalidate caches
    cache.Delete(fmt.Sprintf("comments:%d", postID))
    cache.Delete(fmt.Sprintf("post:%d", postID)) // Comment count changed

    respondJSON(w, http.StatusCreated, comment)
}

func getCacheStats(w http.ResponseWriter, r *http.Request) {
    metrics := cache.GetMetrics()
    respondJSON(w, http.StatusOK, metrics)
}

func warmCache(w http.ResponseWriter, r *http.Request) {
    warmed := 0

    storageMu.RLock()
    defer storageMu.RUnlock()

    // Cache all posts
    for id, post := range posts {
        cacheKey := fmt.Sprintf("post:%d", id)
        cache.Set(cacheKey, post, 10*time.Minute)
        warmed++
    }

    // Cache post list
    postList := make([]Post, 0, len(posts))
    for _, post := range posts {
        postList = append(postList, post)
    }
    cache.Set("posts:all", postList, 5*time.Minute)
    warmed++

    // Cache comments
    for postID, commentList := range comments {
        cacheKey := fmt.Sprintf("comments:%d", postID)
        cache.Set(cacheKey, commentList, 3*time.Minute)
        warmed++
    }

    respondJSON(w, http.StatusOK, map[string]interface{}{
        "message":       "Cache warmed successfully",
        "items_cached":  warmed,
        "posts_cached":  len(posts),
        "comment_lists": len(comments),
    })
}

func clearCache(w http.ResponseWriter, r *http.Request) {
    err := cache.Clear()
    if err != nil {
        respondError(w, http.StatusInternalServerError, "Failed to clear cache")
        return
    }

    respondJSON(w, http.StatusOK, map[string]string{
        "message": "Cache cleared successfully",
    })
}

// =============================================================================
// HELPERS
// =============================================================================

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
        Content:      "Go is a great language for building APIs...",
        Author:       "john",
        ViewCount:    0,
        CommentCount: 0,
        Tags:         []string{"golang", "tutorial"},
        CreatedAt:    now,
        UpdatedAt:    now,
    }

    posts[2] = Post{
        ID:           2,
        Title:        "Redis Caching Strategies",
        Content:      "Redis is a fast key-value store...",
        Author:       "jane",
        ViewCount:    0,
        CommentCount: 0,
        Tags:         []string{"redis", "cache"},
        CreatedAt:    now,
        UpdatedAt:    now,
    }

    posts[3] = Post{
        ID:           3,
        Title:        "API Best Practices",
        Content:      "Building production-ready APIs requires...",
        Author:       "bob",
        ViewCount:    0,
        CommentCount: 0,
        Tags:         []string{"api", "best-practices"},
        CreatedAt:    now,
        UpdatedAt:    now,
    }

    nextPostID = 4
}

func warmCacheOnStartup() {
    fmt.Println("Warming cache on startup...")

    storageMu.RLock()
    defer storageMu.RUnlock()

    // Cache all posts
    for id, post := range posts {
        cacheKey := fmt.Sprintf("post:%d", id)
        cache.Set(cacheKey, post, 10*time.Minute)
    }

    // Cache post list
    postList := make([]Post, 0, len(posts))
    for _, post := range posts {
        postList = append(postList, post)
    }
    cache.Set("posts:all", postList, 5*time.Minute)

    fmt.Printf("Cache warmed: %d posts\n", len(posts))
}

// =============================================================================
// MAIN
// =============================================================================

func main() {
    seedDatabase()

    // Initialize Redis cache
    fmt.Println("Connecting to Redis...")
    cache = NewRedisCache("localhost:6379")
    fmt.Println("Connected to Redis successfully")

    // Warm cache on startup
    warmCacheOnStartup()

    r := mux.NewRouter()

    // Post routes
    r.HandleFunc("/posts", getPosts).Methods("GET")
    r.HandleFunc("/posts/{id}", getPost).Methods("GET")
    r.HandleFunc("/posts", createPost).Methods("POST")
    r.HandleFunc("/posts/{id}", updatePost).Methods("PUT")
    r.HandleFunc("/posts/{id}", deletePost).Methods("DELETE")

    // Comment routes
    r.HandleFunc("/posts/{id}/comments", getComments).Methods("GET")
    r.HandleFunc("/posts/{id}/comments", createComment).Methods("POST")

    // Admin routes
    r.HandleFunc("/stats/cache", getCacheStats).Methods("GET")
    r.HandleFunc("/cache/warm", warmCache).Methods("POST")
    r.HandleFunc("/cache/clear", clearCache).Methods("DELETE")

    fmt.Println("\nServer starting on :8080")
    fmt.Println("Make sure Redis is running:")
    fmt.Println("  docker run -d -p 6379:6379 redis:latest")
    fmt.Println("\nEndpoints:")
    fmt.Println("  GET    /posts")
    fmt.Println("  GET    /posts/{id}")
    fmt.Println("  POST   /posts")
    fmt.Println("  PUT    /posts/{id}")
    fmt.Println("  DELETE /posts/{id}")
    fmt.Println("  GET    /posts/{id}/comments")
    fmt.Println("  POST   /posts/{id}/comments")
    fmt.Println("  GET    /stats/cache")
    fmt.Println("  POST   /cache/warm")
    fmt.Println("  DELETE /cache/clear")

    http.ListenAndServe(":8080", r)
}
```

---

## Key Concepts Explained

### 1. Cache Stampede Prevention

```go
func (c *RedisCache) GetOrLoad(key string, dest interface{}, ttl time.Duration,
    loader func() (interface{}, error)) error {

    // 1. Try cache first
    err := c.Get(key, dest)
    if err == nil {
        return nil // Cache hit
    }

    // 2. Get mutex for this specific key
    muInterface, _ := c.mu.LoadOrStore(key, &sync.Mutex{})
    mu := muInterface.(*sync.Mutex)

    mu.Lock()
    defer mu.Unlock()

    // 3. Double-check cache (another goroutine might have loaded it)
    err = c.Get(key, dest)
    if err == nil {
        atomic.AddInt64(&c.stampedePrevented, 1)
        return nil // Stampede prevented!
    }

    // 4. Load from source
    data, err := loader()
    if err != nil {
        return err
    }

    // 5. Cache and return
    c.Set(key, data, ttl)
    // ...
}
```

**How it prevents stampede**:
- When cache expires on popular item
- 100 concurrent requests arrive
- Request 1 acquires mutex for that key
- Requests 2-100 wait on same mutex
- Request 1 loads data, caches it
- Requests 2-100 get data from cache (not DB!)
- Result: 1 DB query instead of 100

### 2. Write-Through Caching

```go
func createPost() {
    // 1. Write to database
    posts[id] = post

    // 2. Write to cache immediately
    cache.Set(fmt.Sprintf("post:%d", id), post, 10*time.Minute)

    // 3. Invalidate related caches
    cache.Delete("posts:all")
}
```

**Benefits**:
- Cache always has latest data
- No stale data issues
- Next read is immediate cache hit

### 3. Pattern-Based Deletion with SCAN

```go
func (c *RedisCache) DeletePattern(pattern string) error {
    // Use SCAN to find matching keys (safe for production)
    iter := c.client.Scan(ctx, 0, pattern, 0).Iterator()

    keys := []string{}
    for iter.Next(ctx) {
        keys = append(keys, iter.Val())
    }

    // Delete all matches
    if len(keys) > 0 {
        return c.client.Del(ctx, keys...).Err()
    }

    return nil
}
```

**Why SCAN not KEYS**:
- `KEYS` blocks Redis (bad in production)
- `SCAN` is cursor-based (doesn't block)
- Safe to use with large datasets

### 4. Cache Warming

```go
func warmCacheOnStartup() {
    // Pre-load popular data
    for id, post := range posts {
        cache.Set(fmt.Sprintf("post:%d", id), post, 10*time.Minute)
    }

    // Pre-load lists
    cache.Set("posts:all", postList, 5*time.Minute)
}
```

**Prevents**:
- Cold start slowness
- Stampede on restart
- Poor initial performance

### 5. Metrics Tracking

```go
type RedisCache struct {
    hits              int64  // Atomic counter
    misses            int64  // Atomic counter
    stampedePrevented int64  // Atomic counter
    totalLatency      int64  // Microseconds
    requests          int64  // Total requests
}

// Track on every Get
func (c *RedisCache) Get(key string, dest interface{}) error {
    start := time.Now()

    // ... get from Redis ...

    latency := time.Since(start).Microseconds()
    atomic.AddInt64(&c.totalLatency, latency)
    atomic.AddInt64(&c.requests, 1)

    if found {
        atomic.AddInt64(&c.hits, 1)
    } else {
        atomic.AddInt64(&c.misses, 1)
    }
}
```

**Metrics available**:
- Hit rate (should be >80%)
- Average latency
- Stampede prevention count
- Cache size

---

## Performance Comparison

### Without Stampede Prevention
```
Cache expires on popular post
100 concurrent requests arrive

Request 1-100: All hit database simultaneously
Database: 100 queries at once (overload!)
Response time: 100ms each
Total DB time: 10 seconds of queries
```

### With Stampede Prevention
```
Cache expires on popular post
100 concurrent requests arrive

Request 1: Acquires mutex, loads from DB (100ms)
Request 2-100: Wait on mutex, then get from cache (<1ms each)
Database: 1 query total
Response time: Request 1: 100ms, Others: ~100ms wait + <1ms
Total DB time: 100ms
Stampede prevented: 99
```

**Speedup**: 100x fewer database queries!

---

## Testing the Solution

### Start Redis
```bash
docker run -d -p 6379:6379 redis:latest
```

### Test Stampede Prevention
```bash
# Simulate high concurrency (10 parallel requests)
seq 10 | xargs -n1 -P10 curl -s http://localhost:8080/posts/1 > /dev/null

# Check stats
curl http://localhost:8080/stats/cache
```

**Expected output**:
```json
{
  "hits": 9,
  "misses": 1,
  "stampede_prevented": 9,
  "hit_rate": 90.0
}
```

### Test Write-Through
```bash
# Create post (write-through caching)
curl -X POST http://localhost:8080/posts \
  -H "Content-Type: application/json" \
  -d '{"title":"Test Post","content":"Content","author":"alice","tags":["test"]}'

# Immediately read - should be cached
curl http://localhost:8080/posts/4
```

### Test Cache Warming
```bash
# Clear cache
curl -X DELETE http://localhost:8080/cache/clear

# Check stats (should be 0 hits)
curl http://localhost:8080/stats/cache

# Warm cache
curl -X POST http://localhost:8080/cache/warm

# Read posts - all should be HITs
curl http://localhost:8080/posts/1
curl http://localhost:8080/posts/2
curl http://localhost:8080/posts/3

# Check stats
curl http://localhost:8080/stats/cache
```

### Test Pattern Deletion
```bash
# Cache multiple items
curl http://localhost:8080/posts/1
curl http://localhost:8080/posts/2
curl http://localhost:8080/posts/1/comments

# Delete all post caches
# In Redis CLI: SCAN 0 MATCH post:*
# Then: DEL post:1 post:2

# Or restart app and check /stats/cache
```

---

## Redis Commands Reference

### View Cache in Redis CLI
```bash
# Connect to Redis
docker exec -it <container-id> redis-cli

# List all keys
KEYS *

# Get a value
GET post:1

# Check TTL
TTL post:1

# Delete a key
DEL post:1

# Delete by pattern (use SCAN in production)
SCAN 0 MATCH post:*

# Clear all
FLUSHDB

# Check database size
DBSIZE
```

---

## What You've Learned

✅ **Redis integration** with go-redis client  
✅ **Cache stampede prevention** using per-key mutexes  
✅ **Write-through pattern** for consistency  
✅ **Cache warming** to prevent cold starts  
✅ **Pattern-based invalidation** with SCAN  
✅ **Distributed caching** across multiple servers  
✅ **Metrics tracking** for monitoring  
✅ **Production-ready patterns** for high-traffic APIs  

You now understand enterprise-level caching with Redis!
