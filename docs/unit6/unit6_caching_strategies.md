# Unit 6: Caching Strategies

**Duration**: 60-75 minutes  
**Prerequisites**: Units 1-5 (Go fundamentals, HTTP servers, Authentication, Versioning, Documentation)  
**Goal**: Improve API performance with effective caching strategies

---

## 6.1 Why Caching?

### The Problem: Expensive Operations

Without caching:
```
User Request → API → Database Query (100ms) → Response
User Request → API → Database Query (100ms) → Response
User Request → API → Database Query (100ms) → Response
```

**Every request hits the database** - slow and expensive.

### The Solution: Caching

With caching:
```
User Request → API → Check Cache (1ms) → Response (from cache)
User Request → API → Check Cache (1ms) → Response (from cache)
User Request → API → Check Cache (1ms) → Database (100ms) → Update Cache → Response
```

**Most requests served from cache** - fast and cheap.

### Benefits
- ⚡ **Faster response times** (1-10ms vs 50-200ms)
- 💰 **Reduced database load** (fewer queries)
- 📈 **Better scalability** (handle more traffic)
- 💵 **Lower costs** (fewer database resources needed)

---

## 6.2 When to Cache?

### ✅ Good Candidates for Caching

1. **Frequently accessed data**
   - User profiles
   - Product catalogs
   - Configuration settings

2. **Expensive to compute**
   - Complex aggregations
   - Reports and analytics
   - Search results

3. **Rarely changes**
   - Static content
   - Reference data
   - Historical data

4. **Same data for many users**
   - Product listings
   - Blog posts
   - Public API responses

### ❌ Poor Candidates for Caching

1. **Highly personalized data**
   - Shopping carts (different per user)
   - User-specific feeds

2. **Frequently changing data**
   - Stock prices (real-time)
   - Live scores
   - Inventory counts

3. **Security-sensitive data**
   - Passwords
   - Payment details
   - Private user data

4. **One-time data**
   - Unique transactions
   - Single-use tokens

---

## 6.3 Caching Strategies

### Strategy 1: Cache-Aside (Lazy Loading)

**Most common pattern**

```go
func GetUser(id int) (*User, error) {
    // 1. Try to get from cache
    cached, err := cache.Get(fmt.Sprintf("user:%d", id))
    if err == nil {
        // Cache hit
        var user User
        json.Unmarshal(cached, &user)
        return &user, nil
    }
    
    // 2. Cache miss - get from database
    user, err := db.QueryUser(id)
    if err != nil {
        return nil, err
    }
    
    // 3. Store in cache for next time
    data, _ := json.Marshal(user)
    cache.Set(fmt.Sprintf("user:%d", id), data, 5*time.Minute)
    
    return user, nil
}
```

**Pros**:
- Only cache what's needed
- Application controls cache logic
- Cache failure doesn't break app

**Cons**:
- Initial request is slow (cache miss)
- Cache and database can get out of sync

**Best for**: Read-heavy workloads

---

### Strategy 2: Write-Through

**Update cache and database together**

```go
func UpdateUser(id int, user User) error {
    // 1. Update database
    err := db.UpdateUser(id, user)
    if err != nil {
        return err
    }
    
    // 2. Update cache immediately
    data, _ := json.Marshal(user)
    cache.Set(fmt.Sprintf("user:%d", id), data, 5*time.Minute)
    
    return nil
}
```

**Pros**:
- Cache always up-to-date
- No stale data issues

**Cons**:
- Write operations slower (2 operations)
- Wastes cache space if data not read

**Best for**: Write-heavy workloads with consistent reads

---

### Strategy 3: Write-Behind (Write-Back)

**Update cache first, database later**

```go
func UpdateUser(id int, user User) error {
    // 1. Update cache immediately
    data, _ := json.Marshal(user)
    cache.Set(fmt.Sprintf("user:%d", id), data, 5*time.Minute)
    
    // 2. Queue database update for later
    queue.Push(DatabaseUpdate{
        Type: "user",
        ID:   id,
        Data: user,
    })
    
    return nil
}

// Background worker processes queue
func databaseWorker() {
    for update := range queue {
        db.UpdateUser(update.ID, update.Data)
    }
}
```

**Pros**:
- Very fast writes
- Batching possible

**Cons**:
- Risk of data loss if cache fails
- Complex to implement

**Best for**: High-write workloads, acceptable data loss risk

---

### Strategy 4: Read-Through

**Cache handles database reads**

```go
// Cache layer automatically loads from database
user, err := cache.GetOrLoad("user:1", func() (interface{}, error) {
    return db.QueryUser(1)
})
```

**Pros**:
- Simple application code
- Consistent caching logic

**Cons**:
- Needs smart cache library
- Less control

**Best for**: Simple use cases with caching library

---

## 6.4 In-Memory Caching (Go)

### Using sync.Map

```go
package main

import (
    "encoding/json"
    "sync"
    "time"
)

type CacheItem struct {
    Data      []byte
    ExpiresAt time.Time
}

type MemoryCache struct {
    items sync.Map
}

func NewMemoryCache() *MemoryCache {
    cache := &MemoryCache{}
    
    // Background cleanup
    go func() {
        ticker := time.NewTicker(1 * time.Minute)
        for range ticker.C {
            cache.cleanup()
        }
    }()
    
    return cache
}

func (c *MemoryCache) Set(key string, value interface{}, ttl time.Duration) error {
    data, err := json.Marshal(value)
    if err != nil {
        return err
    }
    
    item := CacheItem{
        Data:      data,
        ExpiresAt: time.Now().Add(ttl),
    }
    
    c.items.Store(key, item)
    return nil
}

func (c *MemoryCache) Get(key string) ([]byte, error) {
    val, exists := c.items.Load(key)
    if !exists {
        return nil, errors.New("key not found")
    }
    
    item := val.(CacheItem)
    
    // Check expiration
    if time.Now().After(item.ExpiresAt) {
        c.items.Delete(key)
        return nil, errors.New("key expired")
    }
    
    return item.Data, nil
}

func (c *MemoryCache) Delete(key string) {
    c.items.Delete(key)
}

func (c *MemoryCache) cleanup() {
    c.items.Range(func(key, value interface{}) bool {
        item := value.(CacheItem)
        if time.Now().After(item.ExpiresAt) {
            c.items.Delete(key)
        }
        return true
    })
}
```

**Usage**:
```go
cache := NewMemoryCache()

// Set
user := User{ID: 1, Name: "John"}
cache.Set("user:1", user, 5*time.Minute)

// Get
data, err := cache.Get("user:1")
if err == nil {
    var user User
    json.Unmarshal(data, &user)
}

// Delete
cache.Delete("user:1")
```

**Pros**:
- No external dependencies
- Very fast (in-process)
- Simple to implement

**Cons**:
- Lost on restart
- Not shared across servers
- Limited by RAM

**Best for**: Single-server apps, session data, temporary data

---

## 6.5 Redis Caching

### Setting Up Redis

```bash
# Install Redis
go get github.com/redis/go-redis/v9

# Start Redis (Docker)
docker run -d -p 6379:6379 redis:latest
```

### Basic Redis Usage

```go
package main

import (
    "context"
    "encoding/json"
    "time"
    
    "github.com/redis/go-redis/v9"
)

var ctx = context.Background()

func main() {
    // Connect to Redis
    rdb := redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "",
        DB:       0,
    })
    
    // Ping to check connection
    _, err := rdb.Ping(ctx).Result()
    if err != nil {
        panic(err)
    }
    
    // Set a value
    user := User{ID: 1, Name: "John"}
    data, _ := json.Marshal(user)
    rdb.Set(ctx, "user:1", data, 5*time.Minute)
    
    // Get a value
    val, err := rdb.Get(ctx, "user:1").Result()
    if err == nil {
        var user User
        json.Unmarshal([]byte(val), &user)
    }
    
    // Delete a value
    rdb.Del(ctx, "user:1")
}
```

### Redis Cache Wrapper

```go
type RedisCache struct {
    client *redis.Client
}

func NewRedisCache(addr string) *RedisCache {
    client := redis.NewClient(&redis.Options{
        Addr: addr,
    })
    
    return &RedisCache{client: client}
}

func (c *RedisCache) Set(key string, value interface{}, ttl time.Duration) error {
    data, err := json.Marshal(value)
    if err != nil {
        return err
    }
    
    return c.client.Set(ctx, key, data, ttl).Err()
}

func (c *RedisCache) Get(key string, dest interface{}) error {
    val, err := c.client.Get(ctx, key).Result()
    if err != nil {
        return err
    }
    
    return json.Unmarshal([]byte(val), dest)
}

func (c *RedisCache) Delete(key string) error {
    return c.client.Del(ctx, key).Err()
}

func (c *RedisCache) DeletePattern(pattern string) error {
    // Find all matching keys
    keys, err := c.client.Keys(ctx, pattern).Result()
    if err != nil {
        return err
    }
    
    // Delete all matching keys
    if len(keys) > 0 {
        return c.client.Del(ctx, keys...).Err()
    }
    
    return nil
}
```

**Pros**:
- Shared across servers
- Persists on restart (configurable)
- Rich data structures
- Built-in expiration

**Cons**:
- External dependency
- Network latency
- More complex setup

**Best for**: Multi-server apps, distributed systems, production

---

## 6.6 Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things." - Phil Karlton

### Time-Based Expiration (TTL)

```go
// Set with 5-minute expiration
cache.Set("user:1", user, 5*time.Minute)
```

**Pros**: Simple, automatic cleanup  
**Cons**: May serve stale data

### Manual Invalidation

```go
func UpdateUser(id int, user User) error {
    // Update database
    db.UpdateUser(id, user)
    
    // Invalidate cache
    cache.Delete(fmt.Sprintf("user:%d", id))
    
    return nil
}
```

**Pros**: Always fresh data  
**Cons**: Must remember to invalidate

### Pattern-Based Invalidation

```go
func UpdateUser(id int, user User) error {
    db.UpdateUser(id, user)
    
    // Invalidate all user-related caches
    cache.DeletePattern("user:*")
    cache.DeletePattern("user_list:*")
    
    return nil
}
```

**Pros**: Handles related data  
**Cons**: May invalidate too much

### Event-Based Invalidation

```go
type CacheInvalidationEvent struct {
    Key string
}

func UpdateUser(id int, user User) error {
    db.UpdateUser(id, user)
    
    // Publish event
    eventBus.Publish(CacheInvalidationEvent{
        Key: fmt.Sprintf("user:%d", id),
    })
    
    return nil
}

// Listener
func onInvalidation(event CacheInvalidationEvent) {
    cache.Delete(event.Key)
}
```

**Pros**: Decoupled, flexible  
**Cons**: More complex

---

## 6.7 Cache Key Design

### Best Practices

**Use namespaces**:
```go
// Good
"user:123"
"product:456"
"session:abc123"

// Bad
"123"
"456"
"abc123"
```

**Include version**:
```go
"v1:user:123"
"v2:user:123"  // Can coexist during migration
```

**Be specific**:
```go
// Good
"user:123:profile"
"user:123:settings"

// Too general
"user:123"  // What data?
```

**Use consistent format**:
```go
// Choose one format and stick to it
"resource:id:subresource"
"user:123:posts"
"user:456:comments"
```

---

## 6.8 Cache Warming

### Cold Start Problem

After deployment, cache is empty → all requests hit database → slow responses.

### Solution: Pre-populate Cache

```go
func WarmCache() {
    log.Println("Warming cache...")
    
    // Load frequently accessed data
    products, _ := db.GetTopProducts(100)
    for _, product := range products {
        cache.Set(fmt.Sprintf("product:%d", product.ID), product, 1*time.Hour)
    }
    
    // Load configuration
    config, _ := db.GetConfig()
    cache.Set("config", config, 24*time.Hour)
    
    log.Println("Cache warmed")
}

func main() {
    // Connect to cache
    cache = NewRedisCache("localhost:6379")
    
    // Warm cache on startup
    WarmCache()
    
    // Start server
    http.ListenAndServe(":8080", router)
}
```

---

## 6.9 Complete Example: Cached API

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strconv"
    "time"
    
    "github.com/gorilla/mux"
    "github.com/redis/go-redis/v9"
)

type Product struct {
    ID          int     `json:"id"`
    Name        string  `json:"name"`
    Price       float64 `json:"price"`
    Description string  `json:"description"`
}

var (
    cache *RedisCache
    db    *Database
)

func getProduct(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])
    
    cacheKey := fmt.Sprintf("product:%d", id)
    
    // Try cache first
    var product Product
    err := cache.Get(cacheKey, &product)
    if err == nil {
        // Cache hit
        w.Header().Set("X-Cache", "HIT")
        json.NewEncoder(w).Encode(product)
        return
    }
    
    // Cache miss - query database
    product, err = db.GetProduct(id)
    if err != nil {
        http.Error(w, "Product not found", http.StatusNotFound)
        return
    }
    
    // Store in cache
    cache.Set(cacheKey, product, 5*time.Minute)
    
    w.Header().Set("X-Cache", "MISS")
    json.NewEncoder(w).Encode(product)
}

func updateProduct(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])
    
    var product Product
    json.NewDecoder(r.Body).Decode(&product)
    
    // Update database
    err := db.UpdateProduct(id, product)
    if err != nil {
        http.Error(w, "Update failed", http.StatusInternalServerError)
        return
    }
    
    // Invalidate cache
    cacheKey := fmt.Sprintf("product:%d", id)
    cache.Delete(cacheKey)
    
    // Also invalidate list caches
    cache.DeletePattern("products:list:*")
    
    json.NewEncoder(w).Encode(product)
}

func main() {
    // Initialize
    cache = NewRedisCache("localhost:6379")
    db = NewDatabase()
    
    // Warm cache
    WarmCache()
    
    // Routes
    r := mux.NewRouter()
    r.HandleFunc("/products/{id}", getProduct).Methods("GET")
    r.HandleFunc("/products/{id}", updateProduct).Methods("PUT")
    
    http.ListenAndServe(":8080", r)
}
```

---

## 6.10 Monitoring Cache Performance

### Key Metrics

```go
type CacheStats struct {
    Hits   int64
    Misses int64
    Errors int64
}

func (c *RedisCache) Get(key string, dest interface{}) error {
    val, err := c.client.Get(ctx, key).Result()
    if err == redis.Nil {
        atomic.AddInt64(&stats.Misses, 1)
        return err
    } else if err != nil {
        atomic.AddInt64(&stats.Errors, 1)
        return err
    }
    
    atomic.AddInt64(&stats.Hits, 1)
    return json.Unmarshal([]byte(val), dest)
}

func getCacheHitRate() float64 {
    total := stats.Hits + stats.Misses
    if total == 0 {
        return 0
    }
    return float64(stats.Hits) / float64(total) * 100
}
```

**Monitor**:
- **Hit rate**: Should be > 80% for effective caching
- **Miss rate**: High misses = poor cache strategy
- **Eviction rate**: Too high = cache too small
- **Latency**: Cache should be < 10ms

---

## 6.11 Best Practices

### ✅ DO

1. **Cache expensive operations** (database queries, API calls)
2. **Use appropriate TTL** (balance freshness vs hits)
3. **Monitor cache performance** (hit rate, latency)
4. **Handle cache failures gracefully** (fallback to database)
5. **Use consistent key naming** (resource:id:attribute)
6. **Invalidate on updates** (keep cache fresh)
7. **Warm cache on startup** (avoid cold start)
8. **Set size limits** (prevent memory issues)

### ❌ DON'T

1. **Cache everything** (wasted memory)
2. **Use cache as primary database** (cache can fail)
3. **Cache sensitive data** (security risk)
4. **Forget to expire keys** (memory leak)
5. **Use complex cache keys** (hard to invalidate)
6. **Cache highly personalized data** (low hit rate)
7. **Ignore cache failures** (cascading failures)
8. **Skip monitoring** (can't optimize blind)

---

## 6.12 Caching Anti-Patterns

### Cache Stampede

**Problem**: Cache expires → many requests hit database simultaneously.

**Solution**: Use mutex/lock:
```go
var mu sync.Mutex

func getUser(id int) User {
    mu.Lock()
    defer mu.Unlock()
    
    // Double-check cache
    if cached := cache.Get(id); cached != nil {
        return cached
    }
    
    // Load from database
    user := db.Get(id)
    cache.Set(id, user, 5*time.Minute)
    return user
}
```

### Cache Penetration

**Problem**: Queries for non-existent data bypass cache → hit database every time.

**Solution**: Cache null results:
```go
user, err := db.Get(id)
if err != nil {
    // Cache "not found" for 1 minute
    cache.Set(id, nil, 1*time.Minute)
    return nil
}
```

---

## Key Takeaways

✅ **Caching improves performance** dramatically  
✅ **Cache-aside** is the most common pattern  
✅ **In-memory** for single server, **Redis** for distributed  
✅ **TTL** handles automatic expiration  
✅ **Invalidation** is the hardest part  
✅ **Monitor hit rates** to measure effectiveness  
✅ **Handle cache failures** gracefully  
✅ **Cache key design** matters for invalidation  

---

## What's Next?

Unit 7 will cover Pagination strategies for handling large datasets efficiently! 📄
