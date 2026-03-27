# Unit 6 - Exercise 1 Solution: In-Memory Cache for Product API

**Complete implementation with explanations**

---

## Full Solution Code

```go
package main

import (
    "encoding/json"
    "errors"
    "fmt"
    "net/http"
    "strconv"
    "strings"
    "sync"
    "sync/atomic"
    "time"

    "github.com/gorilla/mux"
)

// =============================================================================
// MODELS
// =============================================================================

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
    items  sync.Map
    hits   int64
    misses int64
}

type CacheStats struct {
    Hits    int64   `json:"hits"`
    Misses  int64   `json:"misses"`
    Size    int     `json:"size"`
    HitRate float64 `json:"hit_rate"`
}

// =============================================================================
// STORAGE
// =============================================================================

var (
    products   = make(map[int]Product)
    nextID     = 1
    productsMu sync.RWMutex
    cache      *MemoryCache
)

// =============================================================================
// CACHE IMPLEMENTATION
// =============================================================================

func NewMemoryCache() *MemoryCache {
    cache := &MemoryCache{}

    // Start background cleanup goroutine
    go func() {
        ticker := time.NewTicker(1 * time.Minute)
        defer ticker.Stop()

        for range ticker.C {
            cache.cleanup()
        }
    }()

    return cache
}

func (c *MemoryCache) Set(key string, value interface{}, ttl time.Duration) error {
    // Marshal to JSON
    data, err := json.Marshal(value)
    if err != nil {
        return err
    }

    // Create cache item with expiration
    item := CacheItem{
        Data:      data,
        ExpiresAt: time.Now().Add(ttl),
    }

    // Store in sync.Map
    c.items.Store(key, item)
    return nil
}

func (c *MemoryCache) Get(key string) ([]byte, error) {
    // Load from sync.Map
    val, exists := c.items.Load(key)
    if !exists {
        atomic.AddInt64(&c.misses, 1)
        return nil, errors.New("key not found")
    }

    item := val.(CacheItem)

    // Check expiration
    if time.Now().After(item.ExpiresAt) {
        c.items.Delete(key)
        atomic.AddInt64(&c.misses, 1)
        return nil, errors.New("key expired")
    }

    // Cache hit
    atomic.AddInt64(&c.hits, 1)
    return item.Data, nil
}

func (c *MemoryCache) Delete(key string) {
    c.items.Delete(key)
}

func (c *MemoryCache) DeletePattern(pattern string) {
    // Remove trailing wildcard for prefix matching
    prefix := strings.TrimSuffix(pattern, "*")

    c.items.Range(func(key, value interface{}) bool {
        keyStr := key.(string)

        // Simple prefix matching
        if strings.HasPrefix(keyStr, prefix) {
            c.items.Delete(key)
        }

        return true // Continue iteration
    })
}

func (c *MemoryCache) cleanup() {
    now := time.Now()

    c.items.Range(func(key, value interface{}) bool {
        item := value.(CacheItem)

        // Delete expired items
        if now.After(item.ExpiresAt) {
            c.items.Delete(key)
        }

        return true
    })
}

func (c *MemoryCache) GetStats() CacheStats {
    hits := atomic.LoadInt64(&c.hits)
    misses := atomic.LoadInt64(&c.misses)

    // Calculate hit rate
    total := hits + misses
    hitRate := 0.0
    if total > 0 {
        hitRate = float64(hits) / float64(total) * 100
    }

    // Count items
    size := 0
    c.items.Range(func(key, value interface{}) bool {
        size++
        return true
    })

    return CacheStats{
        Hits:    hits,
        Misses:  misses,
        Size:    size,
        HitRate: hitRate,
    }
}

// =============================================================================
// HANDLERS
// =============================================================================

func getProducts(w http.ResponseWriter, r *http.Request) {
    cacheKey := "products:all"

    // Try cache first
    cached, err := cache.Get(cacheKey)
    if err == nil {
        // Cache hit
        var products []Product
        json.Unmarshal(cached, &products)

        w.Header().Set("X-Cache", "HIT")
        respondJSON(w, http.StatusOK, products)
        return
    }

    // Cache miss - query database
    productsMu.RLock()
    productList := make([]Product, 0, len(products))
    for _, product := range products {
        productList = append(productList, product)
    }
    productsMu.RUnlock()

    // Cache the result
    cache.Set(cacheKey, productList, 5*time.Minute)

    w.Header().Set("X-Cache", "MISS")
    respondJSON(w, http.StatusOK, productList)
}

func getProduct(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    cacheKey := fmt.Sprintf("product:%d", id)

    // Try cache first
    cached, err := cache.Get(cacheKey)
    if err == nil {
        // Cache hit
        var product Product
        json.Unmarshal(cached, &product)

        w.Header().Set("X-Cache", "HIT")
        respondJSON(w, http.StatusOK, product)
        return
    }

    // Cache miss - query database
    productsMu.RLock()
    product, exists := products[id]
    productsMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Product not found")
        return
    }

    // Cache the result
    cache.Set(cacheKey, product, 10*time.Minute)

    w.Header().Set("X-Cache", "MISS")
    respondJSON(w, http.StatusOK, product)
}

func createProduct(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Name        string  `json:"name"`
        Price       float64 `json:"price"`
        Description string  `json:"description"`
        Stock       int     `json:"stock"`
        Category    string  `json:"category"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Validate
    if req.Name == "" {
        respondError(w, http.StatusBadRequest, "Name is required")
        return
    }

    if req.Price <= 0 {
        respondError(w, http.StatusBadRequest, "Price must be positive")
        return
    }

    // Create product
    productsMu.Lock()
    product := Product{
        ID:          nextID,
        Name:        req.Name,
        Price:       req.Price,
        Description: req.Description,
        Stock:       req.Stock,
        Category:    req.Category,
        UpdatedAt:   time.Now(),
    }
    products[nextID] = product
    nextID++
    productsMu.Unlock()

    // Invalidate list cache
    cache.Delete("products:all")

    respondJSON(w, http.StatusCreated, product)
}

func updateProduct(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    var req struct {
        Name        string  `json:"name"`
        Price       float64 `json:"price"`
        Description string  `json:"description"`
        Stock       int     `json:"stock"`
        Category    string  `json:"category"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Update product
    productsMu.Lock()
    product, exists := products[id]
    if !exists {
        productsMu.Unlock()
        respondError(w, http.StatusNotFound, "Product not found")
        return
    }

    product.Name = req.Name
    product.Price = req.Price
    product.Description = req.Description
    product.Stock = req.Stock
    product.Category = req.Category
    product.UpdatedAt = time.Now()
    products[id] = product
    productsMu.Unlock()

    // Invalidate caches
    cache.Delete(fmt.Sprintf("product:%d", id))
    cache.Delete("products:all")

    respondJSON(w, http.StatusOK, product)
}

func deleteProduct(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    productsMu.Lock()
    _, exists := products[id]
    if exists {
        delete(products, id)
    }
    productsMu.Unlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Product not found")
        return
    }

    // Invalidate caches
    cache.Delete(fmt.Sprintf("product:%d", id))
    cache.Delete("products:all")

    w.WriteHeader(http.StatusNoContent)
}

func getCacheStats(w http.ResponseWriter, r *http.Request) {
    stats := cache.GetStats()
    respondJSON(w, http.StatusOK, stats)
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
    productsMu.Lock()
    defer productsMu.Unlock()

    now := time.Now()

    products[1] = Product{
        ID:          1,
        Name:        "Laptop",
        Price:       999.99,
        Description: "High-performance laptop",
        Stock:       10,
        Category:    "Electronics",
        UpdatedAt:   now,
    }

    products[2] = Product{
        ID:          2,
        Name:        "Mouse",
        Price:       29.99,
        Description: "Wireless mouse",
        Stock:       50,
        Category:    "Accessories",
        UpdatedAt:   now,
    }

    products[3] = Product{
        ID:          3,
        Name:        "Keyboard",
        Price:       79.99,
        Description: "Mechanical keyboard",
        Stock:       25,
        Category:    "Accessories",
        UpdatedAt:   now,
    }

    nextID = 4
}

// =============================================================================
// MAIN
// =============================================================================

func main() {
    seedDatabase()

    // Initialize cache
    cache = NewMemoryCache()

    r := mux.NewRouter()

    // Product routes
    r.HandleFunc("/products", getProducts).Methods("GET")
    r.HandleFunc("/products/{id}", getProduct).Methods("GET")
    r.HandleFunc("/products", createProduct).Methods("POST")
    r.HandleFunc("/products/{id}", updateProduct).Methods("PUT")
    r.HandleFunc("/products/{id}", deleteProduct).Methods("DELETE")

    // Stats route
    r.HandleFunc("/stats", getCacheStats).Methods("GET")

    fmt.Println("Server starting on :8080")
    fmt.Println("Endpoints:")
    fmt.Println("  GET    /products")
    fmt.Println("  GET    /products/{id}")
    fmt.Println("  POST   /products")
    fmt.Println("  PUT    /products/{id}")
    fmt.Println("  DELETE /products/{id}")
    fmt.Println("  GET    /stats")
    http.ListenAndServe(":8080", r)
}
```

---

## Key Concepts Explained

### 1. sync.Map for Thread-Safe Cache

```go
type MemoryCache struct {
    items  sync.Map  // Thread-safe map
    hits   int64     // Atomic counter
    misses int64     // Atomic counter
}
```

**Why sync.Map?**
- Built-in concurrency safety
- No need for manual locking
- Optimized for concurrent reads
- Good for dynamic key sets

### 2. Atomic Operations for Counters

```go
// Increment hits atomically (thread-safe)
atomic.AddInt64(&c.hits, 1)

// Read atomically
hits := atomic.LoadInt64(&c.hits)
```

**Why atomic?**
- Thread-safe without locks
- Prevents race conditions
- Better performance than mutex for counters

### 3. TTL Implementation

```go
type CacheItem struct {
    Data      []byte
    ExpiresAt time.Time  // Expiration timestamp
}

// Check expiration on Get
if time.Now().After(item.ExpiresAt) {
    c.items.Delete(key)
    return nil, errors.New("key expired")
}
```

**How it works**:
- Store expiration time with data
- Check on every Get
- Delete if expired
- Background cleanup removes old entries

### 4. Background Cleanup Goroutine

```go
func NewMemoryCache() *MemoryCache {
    cache := &MemoryCache{}

    go func() {
        ticker := time.NewTicker(1 * time.Minute)
        defer ticker.Stop()

        for range ticker.C {
            cache.cleanup()  // Run every minute
        }
    }()

    return cache
}
```

**Purpose**:
- Remove expired items periodically
- Prevents memory leak
- Runs independently in background

### 5. Cache-Aside Pattern

```go
func getProduct(id int) Product {
    // 1. Try cache
    cached, err := cache.Get(key)
    if err == nil {
        return cached  // Cache hit
    }

    // 2. Cache miss - query database
    product := db.GetProduct(id)

    // 3. Store in cache
    cache.Set(key, product, 10*time.Minute)

    return product
}
```

**Flow**:
1. Check cache first
2. If miss, query database
3. Store result in cache
4. Return data

### 6. Cache Invalidation

```go
func updateProduct(id int) {
    // Update database
    db.UpdateProduct(id, product)

    // Invalidate specific product
    cache.Delete(fmt.Sprintf("product:%d", id))

    // Invalidate list cache
    cache.Delete("products:all")
}
```

**When to invalidate**:
- After updates (PUT)
- After deletes (DELETE)
- After creates (POST invalidates lists)

### 7. Pattern-Based Deletion

```go
func (c *MemoryCache) DeletePattern(pattern string) {
    prefix := strings.TrimSuffix(pattern, "*")

    c.items.Range(func(key, value interface{}) bool {
        keyStr := key.(string)
        if strings.HasPrefix(keyStr, prefix) {
            c.items.Delete(key)
        }
        return true
    })
}

// Usage
cache.DeletePattern("product:*")  // Deletes all products
```

**Useful for**:
- Invalidating related caches
- Clearing by category
- Bulk invalidation

### 8. Cache Statistics

```go
func (c *MemoryCache) GetStats() CacheStats {
    hits := atomic.LoadInt64(&c.hits)
    misses := atomic.LoadInt64(&c.misses)

    total := hits + misses
    hitRate := 0.0
    if total > 0 {
        hitRate = float64(hits) / float64(total) * 100
    }

    return CacheStats{
        Hits:    hits,
        Misses:  misses,
        HitRate: hitRate,
    }
}
```

**Metrics tracked**:
- Hits: Successful cache retrievals
- Misses: Cache not found
- Hit rate: Percentage of hits (should be >80%)
- Size: Number of cached items

---

## Performance Comparison

### Without Cache
```
GET /products/1:     100ms (database query)
GET /products/1:     100ms (database query)
GET /products/1:     100ms (database query)
Total:               300ms
Database queries:    3
```

### With Cache
```
GET /products/1:     100ms (database query + cache store) X-Cache: MISS
GET /products/1:     <1ms  (from cache)                   X-Cache: HIT
GET /products/1:     <1ms  (from cache)                   X-Cache: HIT
Total:               ~102ms
Database queries:    1
Speedup:             ~3x faster
```

---

## Testing the Solution

### Test Cache Hit/Miss

```bash
# First request - MISS
curl -i http://localhost:8080/products/1
# X-Cache: MISS

# Second request - HIT
curl -i http://localhost:8080/products/1
# X-Cache: HIT

# Third request - HIT
curl -i http://localhost:8080/products/1
# X-Cache: HIT
```

### Test Cache Invalidation

```bash
# Cache the product
curl http://localhost:8080/products/1

# Update it
curl -X PUT http://localhost:8080/products/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Updated Laptop","price":899.99,"description":"New desc","stock":5,"category":"Electronics"}'

# Next request is MISS (cache was invalidated)
curl -i http://localhost:8080/products/1
# X-Cache: MISS
```

### Test Statistics

```bash
# Make several requests
curl http://localhost:8080/products/1
curl http://localhost:8080/products/1
curl http://localhost:8080/products/1
curl http://localhost:8080/products/2
curl http://localhost:8080/products/2

# Check stats
curl http://localhost:8080/stats
```

**Expected output**:
```json
{
  "hits": 3,
  "misses": 2,
  "size": 2,
  "hit_rate": 60.0
}
```

### Test Expiration

```bash
# Modify TTL to 5 seconds for testing
# In getProduct: cache.Set(cacheKey, product, 5*time.Second)

# Cache product
curl http://localhost:8080/products/1
# X-Cache: MISS

# Immediate request
curl -i http://localhost:8080/products/1
# X-Cache: HIT

# Wait 6 seconds
sleep 6

# Request after expiration
curl -i http://localhost:8080/products/1
# X-Cache: MISS (expired)
```

---

## Common Patterns

### Cache Key Design

```go
// Good patterns
fmt.Sprintf("product:%d", id)              // Single product
"products:all"                             // All products
fmt.Sprintf("products:category:%s", cat)   // By category
fmt.Sprintf("user:%d:cart", userID)        // User's cart

// Bad patterns
fmt.Sprintf("%d", id)                      // Too generic
"data"                                     // Not descriptive
fmt.Sprintf("product_%d_details", id)      // Inconsistent format
```

### TTL Selection

```go
// Fast-changing data
cache.Set(key, data, 1*time.Minute)

// Moderate-changing data
cache.Set(key, data, 5*time.Minute)

// Slow-changing data
cache.Set(key, data, 1*time.Hour)

// Static data
cache.Set(key, data, 24*time.Hour)
```

---

## What You've Learned

✅ **In-memory caching** with sync.Map  
✅ **Cache-aside pattern** (lazy loading)  
✅ **TTL and expiration** with time.Time  
✅ **Background cleanup** with goroutines  
✅ **Atomic operations** for thread-safe counters  
✅ **Cache invalidation** on writes  
✅ **Pattern-based deletion** for bulk invalidation  
✅ **Cache statistics** and monitoring  
✅ **X-Cache headers** for debugging  

You now understand fundamental caching patterns used in production APIs!
