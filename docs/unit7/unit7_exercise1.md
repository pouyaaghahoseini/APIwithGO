# Unit 7 - Exercise 1: Offset Pagination with Filtering

**Difficulty**: Intermediate  
**Estimated Time**: 45-60 minutes  
**Concepts Covered**: Offset pagination, filtering, sorting, validation, metadata

---

## Objective

Implement complete offset-based pagination for a Product API with:
- Page-based navigation
- Filtering by multiple criteria
- Multi-field sorting
- Input validation
- Navigation links
- Performance optimization

---

## Requirements

### Pagination Parameters

| Parameter | Type | Default | Max | Description |
|-----------|------|---------|-----|-------------|
| page | int | 1 | - | Page number (1-indexed) |
| limit | int | 10 | 100 | Items per page |

### Filter Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| category | string | Filter by category |
| min_price | float | Minimum price |
| max_price | float | Maximum price |
| in_stock | bool | Only in-stock items |
| search | string | Search in name/description |

### Sort Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| sort | string | Field to sort by (price, name, created_at) |
| order | string | Sort order (asc, desc) |

### Response Format

```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total_items": 156,
    "total_pages": 8
  },
  "links": {
    "first": "/products?page=1&limit=20",
    "previous": "/products?page=1&limit=20",
    "next": "/products?page=3&limit=20",
    "last": "/products?page=8&limit=20"
  }
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
    "sort"
    "strconv"
    "strings"
    "sync"
    "time"

    "github.com/gorilla/mux"
)

type Product struct {
    ID          int       `json:"id"`
    Name        string    `json:"name"`
    Description string    `json:"description"`
    Price       float64   `json:"price"`
    Category    string    `json:"category"`
    Stock       int       `json:"stock"`
    Tags        []string  `json:"tags"`
    CreatedAt   time.Time `json:"created_at"`
}

type PaginationMeta struct {
    Page       int `json:"page"`
    Limit      int `json:"limit"`
    TotalItems int `json:"total_items"`
    TotalPages int `json:"total_pages"`
}

type Links struct {
    First    string `json:"first,omitempty"`
    Previous string `json:"previous,omitempty"`
    Next     string `json:"next,omitempty"`
    Last     string `json:"last,omitempty"`
}

type PaginatedResponse struct {
    Data       []Product      `json:"data"`
    Pagination PaginationMeta `json:"pagination"`
    Links      Links          `json:"links"`
}

type ProductFilters struct {
    Category string
    MinPrice float64
    MaxPrice float64
    InStock  bool
    Search   string
}

type SortConfig struct {
    Field string
    Order string
}

// Storage
var (
    products   []Product
    productsMu sync.RWMutex
)

// TODO: Implement parsePageParams
func parsePageParams(r *http.Request) (page, limit int) {
    // Parse page and limit from query parameters
    // Set defaults: page=1, limit=10
    // Enforce max limit: 100
    // Validate: page >= 1
}

// TODO: Implement parseFilters
func parseFilters(r *http.Request) ProductFilters {
    // Parse all filter parameters
    // Handle boolean conversion for in_stock
}

// TODO: Implement parseSorting
func parseSorting(r *http.Request) SortConfig {
    // Parse sort and order parameters
    // Handle "-price" format (minus = descending)
    // Default order: asc
}

// TODO: Implement filterProducts
func filterProducts(products []Product, filters ProductFilters) []Product {
    // Apply all filters
    // Return filtered results
}

// TODO: Implement sortProducts
func sortProducts(products []Product, config SortConfig) {
    // Sort by specified field and order
    // Support: price, name, created_at
}

// TODO: Implement buildLinks
func buildLinks(baseURL string, page, limit, totalPages int) Links {
    // Build navigation links
    // first, previous, next, last
}

// TODO: Implement getProducts
func getProducts(w http.ResponseWriter, r *http.Request) {
    // 1. Parse pagination parameters
    // 2. Parse filters
    // 3. Parse sorting
    // 4. Get all products
    // 5. Apply filters
    // 6. Apply sorting
    // 7. Calculate pagination metadata
    // 8. Slice for current page
    // 9. Build links
    // 10. Return paginated response
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

    now := time.Now()
    categories := []string{"Electronics", "Books", "Clothing", "Home", "Sports"}

    // Create 150 products
    for i := 1; i <= 150; i++ {
        product := Product{
            ID:          i,
            Name:        fmt.Sprintf("Product %d", i),
            Description: fmt.Sprintf("Description for product %d", i),
            Price:       float64(i*10) + 0.99,
            Category:    categories[i%len(categories)],
            Stock:       i % 20,
            Tags:        []string{fmt.Sprintf("tag%d", i%5)},
            CreatedAt:   now.Add(-time.Duration(i) * time.Hour),
        }
        products = append(products, product)
    }
}

func main() {
    seedDatabase()

    r := mux.NewRouter()

    // TODO: Register route
    // r.HandleFunc("/products", getProducts).Methods("GET")

    fmt.Println("Server starting on :8080")
    fmt.Println("Try: http://localhost:8080/products?page=1&limit=20")
    http.ListenAndServe(":8080", r)
}
```

---

## Your Tasks

### Task 1: Parse Pagination Parameters

Implement `parsePageParams`:
- Parse `page` and `limit` from query string
- Default: `page=1`, `limit=10`
- Validate: `page >= 1`
- Enforce: `limit <= 100`

### Task 2: Parse Filters

Implement `parseFilters`:
- Parse all filter parameters
- Handle type conversions (string, float, bool)
- Return `ProductFilters` struct

### Task 3: Parse Sorting

Implement `parseSorting`:
- Parse `sort` and `order` parameters
- Support `-price` format (minus = desc)
- Default order: `asc`
- Return `SortConfig` struct

### Task 4: Filter Products

Implement `filterProducts`:
- Apply category filter (exact match)
- Apply price range filter (min/max)
- Apply stock filter (in_stock = true)
- Apply search filter (case-insensitive, name + description)
- Return filtered list

### Task 5: Sort Products

Implement `sortProducts`:
- Sort by field: `price`, `name`, `created_at`
- Support both `asc` and `desc` order
- Use `sort.Slice`

### Task 6: Build Navigation Links

Implement `buildLinks`:
- Generate URLs for first, previous, next, last
- Include all query parameters
- Only include links that are valid (e.g., no "previous" on page 1)

### Task 7: Complete Handler

Implement `getProducts`:
1. Parse all parameters
2. Get products from storage
3. Apply filters
4. Apply sorting
5. Calculate pagination metadata
6. Slice for current page
7. Build response with links

---

## Testing Your Implementation

### Test Basic Pagination

```bash
# First page
curl "http://localhost:8080/products?page=1&limit=10"

# Second page
curl "http://localhost:8080/products?page=2&limit=10"

# Large page size
curl "http://localhost:8080/products?page=1&limit=50"
```

### Test Filtering

```bash
# By category
curl "http://localhost:8080/products?category=Electronics"

# By price range
curl "http://localhost:8080/products?min_price=50&max_price=200"

# In stock only
curl "http://localhost:8080/products?in_stock=true"

# Search
curl "http://localhost:8080/products?search=product%205"

# Combined filters
curl "http://localhost:8080/products?category=Books&min_price=100&in_stock=true"
```

### Test Sorting

```bash
# Sort by price ascending
curl "http://localhost:8080/products?sort=price&order=asc"

# Sort by price descending (two formats)
curl "http://localhost:8080/products?sort=price&order=desc"
curl "http://localhost:8080/products?sort=-price"

# Sort by name
curl "http://localhost:8080/products?sort=name"
```

### Test Combined Features

```bash
# Paginate + filter + sort
curl "http://localhost:8080/products?page=2&limit=20&category=Electronics&sort=price&order=desc"
```

### Test Edge Cases

```bash
# Invalid page (should default to 1)
curl "http://localhost:8080/products?page=0"
curl "http://localhost:8080/products?page=-1"

# Exceeding max limit (should cap at 100)
curl "http://localhost:8080/products?limit=500"

# Beyond last page (should return empty array)
curl "http://localhost:8080/products?page=999"
```

---

## Expected Response Examples

### Page 1
```json
{
  "data": [
    {
      "id": 1,
      "name": "Product 1",
      "price": 10.99,
      "category": "Books"
    }
    // ... 9 more items
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total_items": 150,
    "total_pages": 15
  },
  "links": {
    "first": "/products?page=1&limit=10",
    "next": "/products?page=2&limit=10",
    "last": "/products?page=15&limit=10"
  }
}
```

### Page 2 with Filters
```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total_items": 45,
    "total_pages": 3
  },
  "links": {
    "first": "/products?page=1&limit=20&category=Electronics",
    "previous": "/products?page=1&limit=20&category=Electronics",
    "next": "/products?page=3&limit=20&category=Electronics",
    "last": "/products?page=3&limit=20&category=Electronics"
  }
}
```

---

## Bonus Challenges

### Bonus 1: Response Headers

Add pagination headers:
```go
w.Header().Set("X-Total-Count", strconv.Itoa(totalItems))
w.Header().Set("X-Page", strconv.Itoa(page))
w.Header().Set("X-Per-Page", strconv.Itoa(limit))
w.Header().Set("X-Total-Pages", strconv.Itoa(totalPages))
```

### Bonus 2: Link Header (GitHub Style)

```go
// Link: <url?page=3>; rel="next", <url?page=1>; rel="first"
linkHeader := fmt.Sprintf(`<%s>; rel="next", <%s>; rel="first"`, nextURL, firstURL)
w.Header().Set("Link", linkHeader)
```

### Bonus 3: Multi-Field Sorting

```go
// Sort by category, then price
GET /products?sort=category,price&order=asc,desc
```

### Bonus 4: Field Selection

```go
// Return only specific fields
GET /products?fields=id,name,price
```

### Bonus 5: Performance Stats

Add query performance timing:
```go
{
  "data": [...],
  "pagination": {...},
  "meta": {
    "query_time_ms": 15
  }
}
```

---

## Hints

### Hint 1: Parsing Page Params

```go
func parsePageParams(r *http.Request) (page, limit int) {
    page, _ = strconv.Atoi(r.URL.Query().Get("page"))
    limit, _ = strconv.Atoi(r.URL.Query().Get("limit"))
    
    if page < 1 {
        page = 1
    }
    if limit < 1 {
        limit = 10
    }
    if limit > 100 {
        limit = 100
    }
    
    return page, limit
}
```

### Hint 2: Filtering Logic

```go
func filterProducts(products []Product, filters ProductFilters) []Product {
    result := []Product{}
    
    for _, p := range products {
        // Check all filters
        if filters.Category != "" && p.Category != filters.Category {
            continue
        }
        
        if filters.MinPrice > 0 && p.Price < filters.MinPrice {
            continue
        }
        
        // ... other filters
        
        result = append(result, p)
    }
    
    return result
}
```

### Hint 3: Building Links

```go
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

## What You're Learning

✅ **Offset pagination** implementation  
✅ **Query parameter parsing** and validation  
✅ **Multiple filter criteria** application  
✅ **Multi-field sorting** with order  
✅ **Pagination metadata** calculation  
✅ **Navigation link generation**  
✅ **Edge case handling** (invalid params)  
✅ **Response structure** best practices  

This is the foundation for any paginated API!
