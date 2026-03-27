# Unit 7 - Exercise 1 Solution: Offset Pagination with Filtering

**Complete implementation with explanations**

---

## Full Solution Code

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

// =============================================================================
// MODELS
// =============================================================================

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

// =============================================================================
// STORAGE
// =============================================================================

var (
    products   []Product
    productsMu sync.RWMutex
)

// =============================================================================
// PARSING FUNCTIONS
// =============================================================================

func parsePageParams(r *http.Request) (page, limit int) {
    // Parse page parameter
    page, err := strconv.Atoi(r.URL.Query().Get("page"))
    if err != nil || page < 1 {
        page = 1
    }

    // Parse limit parameter
    limit, err = strconv.Atoi(r.URL.Query().Get("limit"))
    if err != nil || limit < 1 {
        limit = 10 // Default
    }
    if limit > 100 {
        limit = 100 // Max
    }

    return page, limit
}

func parseFilters(r *http.Request) ProductFilters {
    query := r.URL.Query()

    // Parse price filters
    minPrice, _ := strconv.ParseFloat(query.Get("min_price"), 64)
    maxPrice, _ := strconv.ParseFloat(query.Get("max_price"), 64)

    // Parse boolean filter
    inStock := query.Get("in_stock") == "true"

    return ProductFilters{
        Category: query.Get("category"),
        MinPrice: minPrice,
        MaxPrice: maxPrice,
        InStock:  inStock,
        Search:   query.Get("search"),
    }
}

func parseSorting(r *http.Request) SortConfig {
    sortField := r.URL.Query().Get("sort")
    order := r.URL.Query().Get("order")

    // Handle "-price" format (minus indicates descending)
    if strings.HasPrefix(sortField, "-") {
        order = "desc"
        sortField = strings.TrimPrefix(sortField, "-")
    }

    // Default order
    if order == "" {
        order = "asc"
    }

    // Default sort field
    if sortField == "" {
        sortField = "id"
    }

    return SortConfig{
        Field: sortField,
        Order: order,
    }
}

// =============================================================================
// FILTERING AND SORTING
// =============================================================================

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

        // Search filter (case-insensitive)
        if filters.Search != "" {
            searchLower := strings.ToLower(filters.Search)
            nameLower := strings.ToLower(p.Name)
            descLower := strings.ToLower(p.Description)

            if !strings.Contains(nameLower, searchLower) &&
                !strings.Contains(descLower, searchLower) {
                continue
            }
        }

        // All filters passed
        result = append(result, p)
    }

    return result
}

func sortProducts(products []Product, config SortConfig) {
    sort.Slice(products, func(i, j int) bool {
        var less bool

        // Determine ordering based on field
        switch config.Field {
        case "price":
            less = products[i].Price < products[j].Price
        case "name":
            less = products[i].Name < products[j].Name
        case "created_at":
            less = products[i].CreatedAt.Before(products[j].CreatedAt)
        default: // "id"
            less = products[i].ID < products[j].ID
        }

        // Reverse if descending
        if config.Order == "desc" {
            return !less
        }
        return less
    })
}

// =============================================================================
// LINK BUILDING
// =============================================================================

func buildLinks(baseURL string, page, limit, totalPages int, filters ProductFilters, sortConfig SortConfig) Links {
    links := Links{}

    // Helper function to build URL with all parameters
    buildURL := func(p int) string {
        url := fmt.Sprintf("%s?page=%d&limit=%d", baseURL, p, limit)

        // Add filters
        if filters.Category != "" {
            url += fmt.Sprintf("&category=%s", filters.Category)
        }
        if filters.MinPrice > 0 {
            url += fmt.Sprintf("&min_price=%.2f", filters.MinPrice)
        }
        if filters.MaxPrice > 0 {
            url += fmt.Sprintf("&max_price=%.2f", filters.MaxPrice)
        }
        if filters.InStock {
            url += "&in_stock=true"
        }
        if filters.Search != "" {
            url += fmt.Sprintf("&search=%s", filters.Search)
        }

        // Add sorting
        if sortConfig.Field != "id" {
            url += fmt.Sprintf("&sort=%s", sortConfig.Field)
        }
        if sortConfig.Order != "asc" {
            url += fmt.Sprintf("&order=%s", sortConfig.Order)
        }

        return url
    }

    // Build navigation links
    if totalPages > 0 {
        links.First = buildURL(1)
        links.Last = buildURL(totalPages)
    }

    if page > 1 {
        links.Previous = buildURL(page - 1)
    }

    if page < totalPages {
        links.Next = buildURL(page + 1)
    }

    return links
}

// =============================================================================
// HANDLERS
// =============================================================================

func getProducts(w http.ResponseWriter, r *http.Request) {
    // 1. Parse pagination parameters
    page, limit := parsePageParams(r)

    // 2. Parse filters
    filters := parseFilters(r)

    // 3. Parse sorting
    sortConfig := parseSorting(r)

    // 4. Get all products
    productsMu.RLock()
    allProducts := make([]Product, len(products))
    copy(allProducts, products)
    productsMu.RUnlock()

    // 5. Apply filters
    filtered := filterProducts(allProducts, filters)

    // 6. Apply sorting
    sortProducts(filtered, sortConfig)

    // 7. Calculate pagination metadata
    totalItems := len(filtered)
    totalPages := (totalItems + limit - 1) / limit
    if totalPages < 1 {
        totalPages = 1
    }

    // 8. Calculate offset and slice for current page
    offset := (page - 1) * limit
    end := offset + limit

    var pageData []Product
    if offset < totalItems {
        if end > totalItems {
            end = totalItems
        }
        pageData = filtered[offset:end]
    } else {
        pageData = []Product{}
    }

    // 9. Build links
    baseURL := "/products"
    links := buildLinks(baseURL, page, limit, totalPages, filters, sortConfig)

    // 10. Build response
    response := PaginatedResponse{
        Data: pageData,
        Pagination: PaginationMeta{
            Page:       page,
            Limit:      limit,
            TotalItems: totalItems,
            TotalPages: totalPages,
        },
        Links: links,
    }

    // Add response headers
    w.Header().Set("X-Total-Count", strconv.Itoa(totalItems))
    w.Header().Set("X-Page", strconv.Itoa(page))
    w.Header().Set("X-Per-Page", strconv.Itoa(limit))
    w.Header().Set("X-Total-Pages", strconv.Itoa(totalPages))

    respondJSON(w, http.StatusOK, response)
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

// =============================================================================
// MAIN
// =============================================================================

func main() {
    seedDatabase()

    r := mux.NewRouter()

    // Product routes
    r.HandleFunc("/products", getProducts).Methods("GET")

    fmt.Println("Server starting on :8080")
    fmt.Println("\nExample requests:")
    fmt.Println("  Basic pagination:")
    fmt.Println("    http://localhost:8080/products?page=1&limit=20")
    fmt.Println("\n  With filters:")
    fmt.Println("    http://localhost:8080/products?category=Electronics&min_price=100")
    fmt.Println("\n  With sorting:")
    fmt.Println("    http://localhost:8080/products?sort=price&order=desc")
    fmt.Println("\n  Combined:")
    fmt.Println("    http://localhost:8080/products?page=2&limit=20&category=Books&sort=price")

    http.ListenAndServe(":8080", r)
}
```

---

## Key Concepts Explained

### 1. Parameter Parsing with Validation

```go
func parsePageParams(r *http.Request) (page, limit int) {
    page, err := strconv.Atoi(r.URL.Query().Get("page"))
    if err != nil || page < 1 {
        page = 1  // Default to first page
    }

    limit, err = strconv.Atoi(r.URL.Query().Get("limit"))
    if err != nil || limit < 1 {
        limit = 10  // Default page size
    }
    if limit > 100 {
        limit = 100  // Enforce maximum
    }

    return page, limit
}
```

**Why validation matters**:
- Prevents negative pages
- Prevents excessive page sizes
- Provides sensible defaults
- Protects server resources

### 2. Multiple Filter Application

```go
func filterProducts(products []Product, filters ProductFilters) []Product {
    result := []Product{}

    for _, p := range products {
        // Each filter is a gate
        if filters.Category != "" && p.Category != filters.Category {
            continue  // Doesn't pass category filter
        }

        if filters.MinPrice > 0 && p.Price < filters.MinPrice {
            continue  // Too cheap
        }

        if filters.MaxPrice > 0 && p.Price > filters.MaxPrice {
            continue  // Too expensive
        }

        // Passed all filters
        result = append(result, p)
    }

    return result
}
```

**Filter chain**:
1. Check category
2. Check price range
3. Check stock status
4. Check search terms
5. Only items passing all filters are included

### 3. Multi-Field Sorting

```go
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
            return !less  // Reverse for descending
        }
        return less
    })
}
```

**How it works**:
- `sort.Slice` takes comparison function
- Compare by specified field
- Reverse result for descending order
- Sorts in-place

### 4. Pagination Math

```go
// Calculate total pages
totalItems := 156
limit := 20
totalPages := (totalItems + limit - 1) / limit  // = 8 pages

// Calculate slice boundaries
page := 2
offset := (page - 1) * limit  // = 20 (skip first page)
end := offset + limit          // = 40

// Slice results
pageData := filtered[offset:end]  // Items 20-39
```

**Edge cases handled**:
- Last page with fewer items
- Page beyond data
- Empty results

### 5. Navigation Links with Filters

```go
func buildLinks(baseURL string, page, limit, totalPages int, 
    filters ProductFilters, sortConfig SortConfig) Links {
    
    buildURL := func(p int) string {
        url := fmt.Sprintf("%s?page=%d&limit=%d", baseURL, p, limit)

        // Preserve all filters in links
        if filters.Category != "" {
            url += fmt.Sprintf("&category=%s", filters.Category)
        }
        if filters.MinPrice > 0 {
            url += fmt.Sprintf("&min_price=%.2f", filters.MinPrice)
        }
        // ... more filters

        return url
    }

    // Build navigation
    links := Links{
        First: buildURL(1),
        Last:  buildURL(totalPages),
    }

    if page > 1 {
        links.Previous = buildURL(page - 1)
    }
    if page < totalPages {
        links.Next = buildURL(page + 1)
    }

    return links
}
```

**Important**: Links preserve all query parameters so navigation maintains filters.

---

## Request Flow

### Example Request
```
GET /products?page=2&limit=20&category=Electronics&sort=price&order=desc
```

### Processing Steps

1. **Parse Parameters**
   - page = 2
   - limit = 20
   - category = "Electronics"
   - sort = "price"
   - order = "desc"

2. **Get All Products**
   - Load all 150 products from storage

3. **Apply Filters**
   - Filter by category: 30 Electronics products
   - Result: 30 products

4. **Apply Sorting**
   - Sort by price descending
   - Result: 30 products, highest price first

5. **Calculate Pagination**
   - totalItems = 30
   - totalPages = 2 (30 / 20 = 1.5 → 2)
   - offset = 20 (page 2 starts at item 20)
   - end = 30 (only 10 items on last page)

6. **Slice for Page**
   - Take items [20:30]
   - Result: 10 products

7. **Build Response**
   ```json
   {
     "data": [10 products],
     "pagination": {
       "page": 2,
       "limit": 20,
       "total_items": 30,
       "total_pages": 2
     },
     "links": {
       "first": "/products?page=1&limit=20&category=Electronics&sort=price&order=desc",
       "previous": "/products?page=1&limit=20&category=Electronics&sort=price&order=desc",
       "last": "/products?page=2&limit=20&category=Electronics&sort=price&order=desc"
     }
   }
   ```

---

## Testing the Solution

### Test 1: Basic Pagination

```bash
# First page
curl "http://localhost:8080/products?page=1&limit=10"
```

**Expected**:
```json
{
  "data": [10 products with IDs 1-10],
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

### Test 2: With Filters

```bash
curl "http://localhost:8080/products?category=Electronics&min_price=100&max_price=500"
```

**Expected**:
- Only Electronics products
- Price between 100 and 500
- Filtered total_items (not 150)

### Test 3: With Sorting

```bash
# Ascending
curl "http://localhost:8080/products?sort=price&order=asc"

# Descending (two formats)
curl "http://localhost:8080/products?sort=price&order=desc"
curl "http://localhost:8080/products?sort=-price"
```

**Expected**:
- Items sorted by price
- Lowest first (asc) or highest first (desc)

### Test 4: Edge Cases

```bash
# Invalid page (defaults to 1)
curl "http://localhost:8080/products?page=-1"

# Excessive limit (caps at 100)
curl "http://localhost:8080/products?limit=1000"

# Beyond last page (returns empty)
curl "http://localhost:8080/products?page=999"
```

### Test 5: Combined

```bash
curl "http://localhost:8080/products?page=2&limit=20&category=Books&min_price=50&sort=price&order=desc"
```

**Expected**:
- Page 2 of Books
- Price >= 50
- Sorted by price descending
- 20 items per page
- Links preserve all parameters

---

## Performance Considerations

### Current Implementation (In-Memory)

```go
// 1. Load all products
allProducts := getAllProducts()  // 150 items

// 2. Filter
filtered := filterProducts(allProducts, filters)  // Maybe 30 items

// 3. Sort
sortProducts(filtered, sortConfig)  // Sort 30 items

// 4. Slice
pageData := filtered[offset:end]  // Take 20 items
```

**Time Complexity**: O(n) for filtering + O(n log n) for sorting

### With Database

```sql
-- Much more efficient
SELECT * FROM products
WHERE category = 'Electronics'
  AND price >= 100
  AND price <= 500
ORDER BY price DESC
LIMIT 20 OFFSET 20
```

**Database does**:
- Filtering with indexes (very fast)
- Sorting with indexes (very fast)
- Only returns 20 items (not all data)

---

## Common Patterns

### Pattern 1: Filter Before Paginate

```go
// CORRECT: Filter first, then paginate
filtered := filterProducts(allProducts, filters)
totalItems := len(filtered)
pageData := filtered[offset:end]
```

**Why**: Total items should reflect filtered count.

### Pattern 2: Sort Before Paginate

```go
// CORRECT: Sort before slicing
sortProducts(filtered, sortConfig)
pageData := filtered[offset:end]
```

**Why**: Need consistent ordering across pages.

### Pattern 3: Preserve Query Parameters

```go
// Links maintain filter/sort state
if filters.Category != "" {
    url += fmt.Sprintf("&category=%s", filters.Category)
}
```

**Why**: Users expect navigation to maintain their current view.

---

## What You've Learned

✅ **Offset pagination** with page + limit  
✅ **Parameter parsing** with validation  
✅ **Multiple filter criteria** chaining  
✅ **Multi-field sorting** with direction  
✅ **Pagination metadata** calculation  
✅ **Navigation links** with query preservation  
✅ **Edge case handling** (invalid input, empty results)  
✅ **Response headers** for metadata  
✅ **Filter + Sort + Paginate** pipeline  

You now understand the complete offset pagination pattern used in most REST APIs!
