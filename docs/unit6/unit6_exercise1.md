# Unit 6 - Exercise 1: In-Memory Cache for Product API

**Difficulty**: Intermediate  
**Estimated Time**: 45-60 minutes  
**Concepts Covered**: In-memory caching, cache-aside pattern, TTL, cache invalidation, hit/miss tracking

---

## Objective

Add in-memory caching to a Product API to improve performance. Implement cache-aside pattern with automatic expiration, manual invalidation on updates, and cache statistics tracking.

---

## Requirements

### API Endpoints

| Method | Path | Description | Caching |
|--------|------|-------------|---------|
| GET | /products | List all products | Cache for 5 min |
| GET | /products/{id} | Get single product | Cache for 10 min |
| POST | /products | Create product | Invalidate list cache |
| PUT | /products/{id} | Update product | Invalidate product + list |
| DELETE | /products/{id} | Delete product | Invalidate product + list |
| GET | /stats | Get cache statistics | No cache |

### Cache Requirements

1. **In-memory cache** using `sync.Map`
2. **TTL support** with automatic expiration
3. **Background cleanup** every minute
4. **Cache statistics** (hits, misses, size)
5. **Cache headers** (X-Cache: HIT/MISS)
6. **Manual invalidation** on write operations

### Models

```go
type Product struct {
    ID          int       `json:"id"`
    Name        string    `json:"name"`
    Price       float64   `json:"price"`
    Description string    `json:"description"`
    Stock       int       `json:"stock"`
    Category    string    `json:"category"`
    UpdatedAt   time.Time `json:"updated_at"`
}

type CacheStats struct {
    Hits        int64   `json:"hits"`
    Misses      int64   `json:"misses"`
    Size        int     `json:"size"`
    HitRate     float64 `json:"hit_rate"`
}
```

---

## Starter Code

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "sync"
    "sync/atomic"
    "time"

    "github.com/gorilla/mux"
)

type Product struct {
    ID          int       `json:"id"`
    Name        string    `json:"name"`
    Price       float64   `json:"price"`
    Description string    `json:"description"`
    Stock       int       `json:"stock"`
    Category    string    `json:"category"`
    UpdatedAt   time.Time `json:"updated_at"`
}

type CacheItem struct {
    Data      []byte
    ExpiresAt time.Time
}

type MemoryCache struct {
    items sync.Map
    hits  int64
    misses int64
}

type CacheStats struct {
    Hits    int64   `json:"hits"`
    Misses  int64   `json:"misses"`
    Size    int     `json:"size"`
    HitRate float64 `json:"hit_rate"`
}

// Storage
var (
    products   = make(map[int]Product)
    nextID     = 1
    productsMu sync.RWMutex
    cache      *MemoryCache
)

// TODO: Implement NewMemoryCache
func NewMemoryCache() *MemoryCache {
    // Hint: Create cache and start background cleanup goroutine
}

// TODO: Implement Set
func (c *MemoryCache) Set(key string, value interface{}, ttl time.Duration) error {
    // Hint: Marshal to JSON, create CacheItem with expiration, store in sync.Map
}

// TODO: Implement Get
func (c *MemoryCache) Get(key string) ([]byte, error) {
    // Hint: Load from sync.Map, check expiration, update hits/misses
}

// TODO: Implement Delete
func (c *MemoryCache) Delete(key string) {
    // Hint: Delete from sync.Map
}

// TODO: Implement DeletePattern
func (c *MemoryCache) DeletePattern(pattern string) {
    // Hint: Range over items, match pattern, delete matches
}

// TODO: Implement cleanup
func (c *MemoryCache) cleanup() {
    // Hint: Range over items, delete expired entries
}

// TODO: Implement GetStats
func (c *MemoryCache) GetStats() CacheStats {
    // Hint: Calculate hit rate, count items, return stats
}

// TODO: Implement getProducts with caching
func getProducts(w http.ResponseWriter, r *http.Request) {
    cacheKey := "products:all"
    
    // Try cache first
    // If hit: unmarshal and return with X-Cache: HIT
    // If miss: query database, cache result, return with X-Cache: MISS
}

// TODO: Implement getProduct with caching
func getProduct(w http.ResponseWriter, r *http.Request) {
    // Extract ID
    // Try cache with key "product:{id}"
    // On miss: query database and cache
}

// TODO: Implement createProduct with cache invalidation
func createProduct(w http.ResponseWriter, r *http.Request) {
    // Create product
    // Invalidate "products:all" cache
}

// TODO: Implement updateProduct with cache invalidation
func updateProduct(w http.ResponseWriter, r *http.Request) {
    // Update product
    // Invalidate "product:{id}" and "products:all"
}

// TODO: Implement deleteProduct with cache invalidation
func deleteProduct(w http.ResponseWriter, r *http.Request) {
    // Delete product
    // Invalidate caches
}

// TODO: Implement getCacheStats
func getCacheStats(w http.ResponseWriter, r *http.Request) {
    // Return cache statistics
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
    productsMu.Lock()
    defer productsMu.Unlock()

    products[1] = Product{
        ID:          1,
        Name:        "Laptop",
        Price:       999.99,
        Description: "High-performance laptop",
        Stock:       10,
        Category:    "Electronics",
        UpdatedAt:   time.Now(),
    }

    products[2] = Product{
        ID:          2,
        Name:        "Mouse",
        Price:       29.99,
        Description: "Wireless mouse",
        Stock:       50,
        Category:    "Accessories",
        UpdatedAt:   time.Now(),
    }

    nextID = 3
}

func main() {
    seedDatabase()
    
    // TODO: Initialize cache
    cache = NewMemoryCache()

    r := mux.NewRouter()
    
    // TODO: Register routes
    
    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", r)
}
```

---

## Your Tasks

### Task 1: Implement MemoryCache

Create the cache with:
- `sync.Map` for thread-safe storage
- Hit/miss counters using `atomic` operations
- Background cleanup goroutine (every 1 minute)
- TTL support with expiration checking

### Task 2: Implement Cache Methods

- `Set(key, value, ttl)`: Store with expiration
- `Get(key)`: Retrieve and check expiration
- `Delete(key)`: Remove single key
- `DeletePattern(pattern)`: Remove matching keys
- `GetStats()`: Return cache statistics

### Task 3: Add Caching to Read Endpoints

**GET /products**:
- Cache key: `"products:all"`
- TTL: 5 minutes
- On hit: return cached data
- On miss: query DB, cache, return

**GET /products/{id}**:
- Cache key: `"product:{id}"`
- TTL: 10 minutes
- Add `X-Cache` header (HIT/MISS)

### Task 4: Implement Cache Invalidation

**POST /products**:
- Invalidate `"products:all"`

**PUT /products/{id}**:
- Invalidate `"product:{id}"`
- Invalidate `"products:all"`

**DELETE /products/{id}**:
- Invalidate `"product:{id}"`
- Invalidate `"products:all"`

### Task 5: Add Cache Statistics

Implement `/stats` endpoint returning:
```json
{
  "hits": 150,
  "misses": 50,
  "size": 25,
  "hit_rate": 75.0
}
```

---

## Testing Your Implementation

### Test Cache Hits

```bash
# First request - MISS
curl -i http://localhost:8080/products/1
# Look for: X-Cache: MISS

# Second request - HIT
curl -i http://localhost:8080/products/1
# Look for: X-Cache: HIT
```

### Test Cache Invalidation

```bash
# Cache product
curl http://localhost:8080/products/1

# Update product
curl -X PUT http://localhost:8080/products/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Updated Laptop","price":899.99}'

# Next request should be MISS (cache invalidated)
curl -i http://localhost:8080/products/1
# Should see: X-Cache: MISS
```

### Test Cache Expiration

```bash
# Cache product with 10-second TTL (modify code for testing)
curl http://localhost:8080/products/1

# Wait 11 seconds
sleep 11

# Request again - should be MISS (expired)
curl -i http://localhost:8080/products/1
```

### Test Cache Statistics

```bash
# Make several requests
curl http://localhost:8080/products/1
curl http://localhost:8080/products/1
curl http://localhost:8080/products/2
curl http://localhost:8080/products/2

# Check stats
curl http://localhost:8080/stats
# Expected: hits > misses
```

---

## Expected Behavior

### Cold Start (No Cache)
```
Request 1: GET /products/1 → Database → 100ms → X-Cache: MISS
Request 2: GET /products/1 → Cache    → 1ms   → X-Cache: HIT
Request 3: GET /products/1 → Cache    → 1ms   → X-Cache: HIT
```

### After Update (Invalidation)
```
Update:    PUT /products/1 → Invalidate cache
Request 4: GET /products/1 → Database → 100ms → X-Cache: MISS
Request 5: GET /products/1 → Cache    → 1ms   → X-Cache: HIT
```

### After Expiration (TTL)
```
Wait 10+ minutes...
Request 6: GET /products/1 → Database → 100ms → X-Cache: MISS
```

---

## Bonus Challenges

### Bonus 1: Cache Warming
Implement cache warming on startup:
```go
func warmCache() {
    // Pre-load popular products into cache
}
```

### Bonus 2: Size Limits
Add max cache size (e.g., 1000 items):
- Track item count
- Implement LRU eviction when full

### Bonus 3: Pattern Matching
Improve `DeletePattern` to support wildcards:
```go
cache.DeletePattern("product:*")  // All products
cache.DeletePattern("products:category:*")  // All category caches
```

### Bonus 4: Cache Middleware
Create middleware to cache all GET requests:
```go
func cacheMiddleware(next http.Handler) http.Handler {
    // Auto-cache GET requests
}
```

### Bonus 5: Stale-While-Revalidate
Serve stale data while refreshing:
```go
// If expired but < 1 minute old:
// - Return cached data immediately
// - Refresh cache in background
```

---

## Hints

### Hint 1: Background Cleanup

```go
func NewMemoryCache() *MemoryCache {
    cache := &MemoryCache{}
    
    go func() {
        ticker := time.NewTicker(1 * time.Minute)
        defer ticker.Stop()
        
        for range ticker.C {
            cache.cleanup()
        }
    }()
    
    return cache
}
```

### Hint 2: Atomic Operations

```go
func (c *MemoryCache) Get(key string) ([]byte, error) {
    val, exists := c.items.Load(key)
    if !exists {
        atomic.AddInt64(&c.misses, 1)
        return nil, errors.New("not found")
    }
    
    atomic.AddInt64(&c.hits, 1)
    // ... rest of logic
}
```

### Hint 3: Pattern Matching

```go
func (c *MemoryCache) DeletePattern(pattern string) {
    c.items.Range(func(key, value interface{}) bool {
        keyStr := key.(string)
        
        // Simple prefix matching
        if strings.HasPrefix(keyStr, strings.TrimSuffix(pattern, "*")) {
            c.items.Delete(key)
        }
        
        return true
    })
}
```

### Hint 4: Cache Key Design

```go
// Good key design
fmt.Sprintf("product:%d", id)           // Single product
"products:all"                          // All products
fmt.Sprintf("products:category:%s", cat) // By category
```

---

## What You're Learning

✅ **In-memory caching** with sync.Map  
✅ **Cache-aside pattern** implementation  
✅ **TTL and expiration** handling  
✅ **Cache invalidation** strategies  
✅ **Hit/miss tracking** with atomics  
✅ **Background cleanup** with goroutines  
✅ **Cache headers** for debugging  
✅ **Performance monitoring** with statistics  

This exercise teaches fundamental caching patterns used in production!
