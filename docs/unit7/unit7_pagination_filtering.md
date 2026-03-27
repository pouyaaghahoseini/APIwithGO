# Unit 7: Pagination & Filtering

**Duration**: 60-75 minutes  
**Prerequisites**: Units 1-6 (Go fundamentals, HTTP servers, Authentication, Versioning, Documentation, Caching)  
**Goal**: Handle large datasets efficiently with pagination and filtering

---

## 7.1 Why Pagination?

### The Problem: Large Datasets

Without pagination:
```go
// Returns ALL products (could be millions)
GET /products
Response: [1000000 products...]  // 500MB response!
```

**Issues**:
- 🐌 Slow responses (seconds or minutes)
- 💾 High memory usage (server and client)
- 📶 Network overload
- 💸 Wasted bandwidth
- 😢 Poor user experience

### The Solution: Pagination

With pagination:
```go
// Returns 20 products at a time
GET /products?page=1&limit=20
Response: [20 products...] + metadata
```

**Benefits**:
- ⚡ Fast responses (<100ms)
- 💾 Low memory usage
- 📶 Reasonable bandwidth
- 😊 Good user experience
- 🔄 Client controls data loading

---

## 7.2 Pagination Strategies

### Strategy 1: Offset-Based (Page Number)

**Most common and easiest to implement**

```go
// Request
GET /products?page=2&limit=20

// SQL
SELECT * FROM products 
LIMIT 20 OFFSET 20  -- Skip first 20, get next 20

// Response
{
  "data": [...],
  "page": 2,
  "limit": 20,
  "total_items": 1000,
  "total_pages": 50
}
```

**Pros**:
- ✅ Simple to implement
- ✅ Easy for users to understand
- ✅ Can jump to any page
- ✅ Shows total pages

**Cons**:
- ❌ Slow on large offsets (OFFSET 1000000)
- ❌ Inconsistent when data changes
- ❌ Not efficient for deep pagination

**Best for**: 
- Small to medium datasets
- UIs with page numbers
- Relatively static data

---

### Strategy 2: Cursor-Based (Keyset)

**Most efficient for large datasets**

```go
// First request
GET /products?limit=20

// Response
{
  "data": [...],
  "next_cursor": "eyJpZCI6MjB9",  // Encoded: {"id": 20}
  "has_more": true
}

// Next request
GET /products?cursor=eyJpZCI6MjB9&limit=20

// SQL
SELECT * FROM products 
WHERE id > 20 
ORDER BY id 
LIMIT 20
```

**Pros**:
- ✅ Consistent results even when data changes
- ✅ Fast at any depth (uses WHERE, not OFFSET)
- ✅ Scales to millions of records
- ✅ Efficient database queries

**Cons**:
- ❌ Can't jump to specific page
- ❌ Can't show total pages
- ❌ More complex to implement
- ❌ Requires sortable field

**Best for**:
- Large datasets
- Real-time feeds
- Infinite scroll UIs
- High-write environments

---

### Strategy 3: Seek Method (Hybrid)

**Combines benefits of both**

```go
// Request
GET /products?after_id=20&limit=20

// SQL
SELECT * FROM products 
WHERE id > 20 
ORDER BY id 
LIMIT 20
```

Similar to cursor-based but more explicit.

---

## 7.3 Implementing Offset Pagination

### Basic Implementation

```go
type PaginatedResponse struct {
    Data       []Product `json:"data"`
    Page       int       `json:"page"`
    Limit      int       `json:"limit"`
    TotalItems int       `json:"total_items"`
    TotalPages int       `json:"total_pages"`
}

func getProducts(w http.ResponseWriter, r *http.Request) {
    // Parse query parameters
    page, _ := strconv.Atoi(r.URL.Query().Get("page"))
    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))
    
    // Default and max values
    if page < 1 {
        page = 1
    }
    if limit < 1 {
        limit = 10
    }
    if limit > 100 {
        limit = 100  // Max page size
    }
    
    // Calculate offset
    offset := (page - 1) * limit
    
    // Query with limit and offset
    products := queryProducts(limit, offset)
    totalItems := countProducts()
    totalPages := (totalItems + limit - 1) / limit
    
    response := PaginatedResponse{
        Data:       products,
        Page:       page,
        Limit:      limit,
        TotalItems: totalItems,
        TotalPages: totalPages,
    }
    
    json.NewEncoder(w).Encode(response)
}

func queryProducts(limit, offset int) []Product {
    // In real app: SELECT * FROM products LIMIT ? OFFSET ?
    productsMu.RLock()
    defer productsMu.RUnlock()
    
    products := []Product{}
    i := 0
    for _, product := range allProducts {
        if i >= offset && len(products) < limit {
            products = append(products, product)
        }
        i++
        if len(products) >= limit {
            break
        }
    }
    
    return products
}
```

### With Navigation Links

```go
type PaginatedResponse struct {
    Data       []Product `json:"data"`
    Page       int       `json:"page"`
    Limit      int       `json:"limit"`
    TotalItems int       `json:"total_items"`
    TotalPages int       `json:"total_pages"`
    Links      Links     `json:"links"`
}

type Links struct {
    First    string `json:"first,omitempty"`
    Previous string `json:"previous,omitempty"`
    Next     string `json:"next,omitempty"`
    Last     string `json:"last,omitempty"`
}

func buildLinks(baseURL string, page, limit, totalPages int) Links {
    links := Links{
        First: fmt.Sprintf("%s?page=1&limit=%d", baseURL, limit),
        Last:  fmt.Sprintf("%s?page=%d&limit=%d", baseURL, totalPages, limit),
    }
    
    if page > 1 {
        links.Previous = fmt.Sprintf("%s?page=%d&limit=%d", baseURL, page-1, limit)
    }
    
    if page < totalPages {
        links.Next = fmt.Sprintf("%s?page=%d&limit=%d", baseURL, page+1, limit)
    }
    
    return links
}
```

---

## 7.4 Implementing Cursor Pagination

### Cursor Structure

```go
type Cursor struct {
    ID        int       `json:"id"`
    CreatedAt time.Time `json:"created_at,omitempty"`
}

// Encode cursor to base64
func encodeCursor(c Cursor) string {
    data, _ := json.Marshal(c)
    return base64.StdEncoding.EncodeToString(data)
}

// Decode cursor from base64
func decodeCursor(s string) (Cursor, error) {
    data, err := base64.StdEncoding.DecodeString(s)
    if err != nil {
        return Cursor{}, err
    }
    
    var cursor Cursor
    err = json.Unmarshal(data, &cursor)
    return cursor, err
}
```

### Implementation

```go
type CursorPaginatedResponse struct {
    Data       []Product `json:"data"`
    NextCursor string    `json:"next_cursor,omitempty"`
    HasMore    bool      `json:"has_more"`
}

func getProducts(w http.ResponseWriter, r *http.Request) {
    cursorStr := r.URL.Query().Get("cursor")
    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))
    
    if limit < 1 {
        limit = 20
    }
    if limit > 100 {
        limit = 100
    }
    
    var afterID int
    if cursorStr != "" {
        cursor, err := decodeCursor(cursorStr)
        if err != nil {
            http.Error(w, "Invalid cursor", http.StatusBadRequest)
            return
        }
        afterID = cursor.ID
    }
    
    // Query with cursor
    products := queryProductsAfter(afterID, limit+1)
    
    hasMore := len(products) > limit
    if hasMore {
        products = products[:limit]
    }
    
    var nextCursor string
    if hasMore && len(products) > 0 {
        lastProduct := products[len(products)-1]
        nextCursor = encodeCursor(Cursor{ID: lastProduct.ID})
    }
    
    response := CursorPaginatedResponse{
        Data:       products,
        NextCursor: nextCursor,
        HasMore:    hasMore,
    }
    
    json.NewEncoder(w).Encode(response)
}

func queryProductsAfter(afterID, limit int) []Product {
    // In real app: SELECT * FROM products WHERE id > ? ORDER BY id LIMIT ?
    productsMu.RLock()
    defer productsMu.RUnlock()
    
    products := []Product{}
    for _, product := range allProducts {
        if product.ID > afterID {
            products = append(products, product)
            if len(products) >= limit {
                break
            }
        }
    }
    
    return products
}
```

---

## 7.5 Filtering

### Query Parameters for Filtering

```go
GET /products?category=electronics&min_price=100&max_price=500&in_stock=true
```

### Implementation

```go
type ProductFilters struct {
    Category  string
    MinPrice  float64
    MaxPrice  float64
    InStock   bool
    Search    string
    Tags      []string
}

func parseFilters(r *http.Request) ProductFilters {
    query := r.URL.Query()
    
    minPrice, _ := strconv.ParseFloat(query.Get("min_price"), 64)
    maxPrice, _ := strconv.ParseFloat(query.Get("max_price"), 64)
    inStock := query.Get("in_stock") == "true"
    
    return ProductFilters{
        Category: query.Get("category"),
        MinPrice: minPrice,
        MaxPrice: maxPrice,
        InStock:  inStock,
        Search:   query.Get("search"),
        Tags:     query["tags"],  // Multiple values
    }
}

func filterProducts(products []Product, filters ProductFilters) []Product {
    result := []Product{}
    
    for _, p := range products {
        // Category filter
        if filters.Category != "" && p.Category != filters.Category {
            continue
        }
        
        // Price range filter
        if filters.MinPrice > 0 && p.Price < filters.MinPrice {
            continue
        }
        if filters.MaxPrice > 0 && p.Price > filters.MaxPrice {
            continue
        }
        
        // Stock filter
        if filters.InStock && p.Stock <= 0 {
            continue
        }
        
        // Search filter
        if filters.Search != "" {
            searchLower := strings.ToLower(filters.Search)
            if !strings.Contains(strings.ToLower(p.Name), searchLower) &&
               !strings.Contains(strings.ToLower(p.Description), searchLower) {
                continue
            }
        }
        
        // Tag filter
        if len(filters.Tags) > 0 {
            hasTag := false
            for _, filterTag := range filters.Tags {
                for _, productTag := range p.Tags {
                    if productTag == filterTag {
                        hasTag = true
                        break
                    }
                }
            }
            if !hasTag {
                continue
            }
        }
        
        result = append(result, p)
    }
    
    return result
}
```

---

## 7.6 Sorting

### Query Parameters

```go
GET /products?sort=price&order=desc
GET /products?sort=-price  // Minus sign indicates descending
```

### Implementation

```go
type SortConfig struct {
    Field string
    Order string  // "asc" or "desc"
}

func parseSorting(r *http.Request) SortConfig {
    sort := r.URL.Query().Get("sort")
    order := r.URL.Query().Get("order")
    
    // Handle "-price" format
    if strings.HasPrefix(sort, "-") {
        order = "desc"
        sort = strings.TrimPrefix(sort, "-")
    }
    
    if order == "" {
        order = "asc"
    }
    
    return SortConfig{
        Field: sort,
        Order: order,
    }
}

func sortProducts(products []Product, config SortConfig) {
    sort.Slice(products, func(i, j int) bool {
        var less bool
        
        switch config.Field {
        case "price":
            less = products[i].Price < products[j].Price
        case "name":
            less = products[i].Name < products[j].Name
        case "created_at":
            less = products[i].CreatedAt.Before(products[j].CreatedAt)
        default:
            less = products[i].ID < products[j].ID
        }
        
        if config.Order == "desc" {
            return !less
        }
        return less
    })
}
```

---

## 7.7 Complete Example: Paginated, Filtered, Sorted API

```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    // 1. Parse pagination
    page, _ := strconv.Atoi(r.URL.Query().Get("page"))
    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))
    
    if page < 1 {
        page = 1
    }
    if limit < 1 {
        limit = 10
    }
    if limit > 100 {
        limit = 100
    }
    
    // 2. Parse filters
    filters := parseFilters(r)
    
    // 3. Parse sorting
    sortConfig := parseSorting(r)
    
    // 4. Get all products
    productsMu.RLock()
    allProducts := make([]Product, 0, len(products))
    for _, p := range products {
        allProducts = append(allProducts, p)
    }
    productsMu.RUnlock()
    
    // 5. Apply filters
    filtered := filterProducts(allProducts, filters)
    
    // 6. Apply sorting
    sortProducts(filtered, sortConfig)
    
    // 7. Calculate pagination
    totalItems := len(filtered)
    totalPages := (totalItems + limit - 1) / limit
    offset := (page - 1) * limit
    
    // 8. Apply pagination
    end := offset + limit
    if end > totalItems {
        end = totalItems
    }
    
    var pageData []Product
    if offset < totalItems {
        pageData = filtered[offset:end]
    } else {
        pageData = []Product{}
    }
    
    // 9. Build response
    response := PaginatedResponse{
        Data:       pageData,
        Page:       page,
        Limit:      limit,
        TotalItems: totalItems,
        TotalPages: totalPages,
    }
    
    json.NewEncoder(w).Encode(response)
}
```

---

## 7.8 Performance Optimization

### 1. Database Indexing

```sql
-- Index for cursor pagination
CREATE INDEX idx_products_id ON products(id);

-- Index for filtering
CREATE INDEX idx_products_category ON products(category);
CREATE INDEX idx_products_price ON products(price);

-- Composite index for common queries
CREATE INDEX idx_products_category_price ON products(category, price);
```

### 2. Caching Paginated Results

```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    // Build cache key from all parameters
    cacheKey := fmt.Sprintf("products:page=%d:limit=%d:category=%s",
        page, limit, filters.Category)
    
    // Try cache
    var response PaginatedResponse
    if cache.Get(cacheKey, &response) == nil {
        json.NewEncoder(w).Encode(response)
        return
    }
    
    // Query and cache
    response = buildPaginatedResponse(page, limit, filters)
    cache.Set(cacheKey, response, 5*time.Minute)
    
    json.NewEncoder(w).Encode(response)
}
```

### 3. Count Optimization

```go
// Expensive: Count on every request
totalItems := countProductsWithFilters(filters)

// Better: Cache the count
cacheKey := fmt.Sprintf("products:count:%s", filtersHash)
if cached := cache.Get(cacheKey); cached != nil {
    totalItems = cached
} else {
    totalItems = countProductsWithFilters(filters)
    cache.Set(cacheKey, totalItems, 10*time.Minute)
}
```

---

## 7.9 Best Practices

### ✅ DO

1. **Set default and maximum limits**
   ```go
   if limit < 1 {
       limit = 10  // Default
   }
   if limit > 100 {
       limit = 100  // Max
   }
   ```

2. **Validate page numbers**
   ```go
   if page < 1 {
       page = 1
   }
   ```

3. **Provide metadata**
   - Total items
   - Total pages
   - Current page
   - Navigation links

4. **Use cursor pagination for large datasets**

5. **Cache paginated results**

6. **Index database columns used in WHERE and ORDER BY**

7. **Document pagination in API docs**

8. **Support multiple sort fields**

### ❌ DON'T

1. **Don't return all data by default**
   - Always paginate

2. **Don't allow unlimited page size**
   - Enforce max limit

3. **Don't use OFFSET for deep pagination**
   - Use cursor pagination instead

4. **Don't forget to handle edge cases**
   - Empty results
   - Last page
   - Invalid parameters

5. **Don't paginate without sorting**
   - Results will be inconsistent

6. **Don't cache without TTL**
   - Stale data issues

---

## 7.10 Common Patterns

### Pattern 1: GitHub-Style Pagination

```go
GET /repos?page=2&per_page=30

{
  "items": [...],
  "total_count": 1234
}

// Headers
Link: <url?page=3>; rel="next", <url?page=1>; rel="prev"
```

### Pattern 2: Relay-Style (GraphQL)

```go
{
  "edges": [
    {
      "node": {...},
      "cursor": "..."
    }
  ],
  "pageInfo": {
    "hasNextPage": true,
    "endCursor": "..."
  }
}
```

### Pattern 3: JSON:API Style

```go
{
  "data": [...],
  "links": {
    "first": "url?page=1",
    "last": "url?page=10",
    "prev": "url?page=1",
    "next": "url?page=3"
  },
  "meta": {
    "total": 200
  }
}
```

---

## 7.11 Client-Side Usage

### Offset Pagination (React Example)

```jsx
function ProductList() {
  const [page, setPage] = useState(1);
  const [data, setData] = useState(null);
  
  useEffect(() => {
    fetch(`/products?page=${page}&limit=20`)
      .then(r => r.json())
      .then(setData);
  }, [page]);
  
  return (
    <>
      {data?.data.map(p => <ProductCard key={p.id} {...p} />)}
      
      <button onClick={() => setPage(page - 1)} disabled={page === 1}>
        Previous
      </button>
      <span>Page {page} of {data?.total_pages}</span>
      <button onClick={() => setPage(page + 1)} disabled={page === data?.total_pages}>
        Next
      </button>
    </>
  );
}
```

### Cursor Pagination (Infinite Scroll)

```jsx
function InfiniteProductList() {
  const [products, setProducts] = useState([]);
  const [cursor, setCursor] = useState(null);
  const [hasMore, setHasMore] = useState(true);
  
  const loadMore = () => {
    const url = cursor 
      ? `/products?cursor=${cursor}&limit=20`
      : `/products?limit=20`;
      
    fetch(url)
      .then(r => r.json())
      .then(data => {
        setProducts([...products, ...data.data]);
        setCursor(data.next_cursor);
        setHasMore(data.has_more);
      });
  };
  
  return (
    <InfiniteScroll onLoadMore={loadMore} hasMore={hasMore}>
      {products.map(p => <ProductCard key={p.id} {...p} />)}
    </InfiniteScroll>
  );
}
```

---

## Key Takeaways

✅ **Pagination prevents slow, large responses**  
✅ **Offset pagination** is simple but slow for deep pages  
✅ **Cursor pagination** is fast and consistent  
✅ **Always set max page size** (e.g., 100)  
✅ **Provide metadata** (total, pages, links)  
✅ **Filter before paginate** for correct totals  
✅ **Cache paginated results** for performance  
✅ **Index database columns** for fast queries  
✅ **Document pagination** in API docs  

---

## What's Next?

Unit 8 will cover Rate Limiting to protect your API from abuse! 🛡️
