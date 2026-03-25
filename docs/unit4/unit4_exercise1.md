# Unit 4 - Exercise 1: E-commerce API Migration (v1 to v2)

**Difficulty**: Intermediate  
**Estimated Time**: 45-60 minutes  
**Concepts Covered**: URL path versioning, breaking changes, model conversion, deprecation headers

---

## Objective

Build an e-commerce API that supports two versions:
- **V1**: Original design with some limitations
- **V2**: Improved design with breaking changes

Learn how to maintain both versions simultaneously while sharing business logic.

---

## Requirements

### V1 API (Original Design)

**Product Model**:
```go
type ProductV1 struct {
    ID          int     `json:"id"`
    Name        string  `json:"name"`
    Price       float64 `json:"price"`
    Description string  `json:"description"`
}
```

**Order Model**:
```go
type OrderV1 struct {
    ID         int     `json:"id"`
    ProductID  int     `json:"product_id"`
    Quantity   int     `json:"quantity"`
    TotalPrice float64 `json:"total_price"`
}
```

### V2 API (Breaking Changes)

**Changes from V1**:
1. ✅ Price split into `price_amount` and `price_currency`
2. ✅ Added `stock` field to products
3. ✅ Added `category` field to products
4. ✅ Order now includes full product details (not just ID)
5. ✅ Added `created_at` timestamp to orders

**Product Model V2**:
```go
type ProductV2 struct {
    ID            int       `json:"id"`
    Name          string    `json:"name"`
    PriceAmount   float64   `json:"price_amount"`
    PriceCurrency string    `json:"price_currency"`
    Description   string    `json:"description"`
    Category      string    `json:"category"`
    Stock         int       `json:"stock"`
    CreatedAt     time.Time `json:"created_at"`
}
```

**Order Model V2**:
```go
type OrderV2 struct {
    ID        int       `json:"id"`
    Product   ProductV2 `json:"product"`  // Full product details
    Quantity  int       `json:"quantity"`
    Total     Money     `json:"total"`
    CreatedAt time.Time `json:"created_at"`
}

type Money struct {
    Amount   float64 `json:"amount"`
    Currency string  `json:"currency"`
}
```

### API Endpoints

| Version | Method | Path | Description |
|---------|--------|------|-------------|
| Both | GET | /api/v1/products | List all products (V1 format) |
| Both | GET | /api/v2/products | List all products (V2 format) |
| Both | GET | /api/v1/products/{id} | Get single product (V1) |
| Both | GET | /api/v2/products/{id} | Get single product (V2) |
| Both | POST | /api/v1/orders | Create order (V1 format) |
| Both | POST | /api/v2/orders | Create order (V2 format) |
| Both | GET | /api/v1/orders/{id} | Get order (V1 format) |
| Both | GET | /api/v2/orders/{id} | Get order (V2 format) |

### Shared Database Model

Create a single database representation that supports both versions:

```go
type ProductRecord struct {
    ID            int
    Name          string
    PriceAmount   float64
    PriceCurrency string
    Description   string
    Category      string
    Stock         int
    CreatedAt     time.Time
}

type OrderRecord struct {
    ID         int
    ProductID  int
    Quantity   int
    TotalAmount float64
    Currency   string
    CreatedAt  time.Time
}
```

### Requirements

1. **V1 Endpoints**: Work with original format
2. **V2 Endpoints**: Work with improved format
3. **Shared Logic**: Business logic shared between versions
4. **Conversion**: Automatic conversion between formats
5. **Deprecation**: V1 should include deprecation headers
6. **Default Currency**: V1 assumes USD, V2 requires explicit currency

---

## Starter Code

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "sync"
    "time"
    
    "github.com/gorilla/mux"
)

// =============================================================================
// V1 MODELS
// =============================================================================

type ProductV1 struct {
    ID          int     `json:"id"`
    Name        string  `json:"name"`
    Price       float64 `json:"price"`
    Description string  `json:"description"`
}

type OrderV1 struct {
    ID         int     `json:"id"`
    ProductID  int     `json:"product_id"`
    Quantity   int     `json:"quantity"`
    TotalPrice float64 `json:"total_price"`
}

// =============================================================================
// V2 MODELS
// =============================================================================

type ProductV2 struct {
    ID            int       `json:"id"`
    Name          string    `json:"name"`
    PriceAmount   float64   `json:"price_amount"`
    PriceCurrency string    `json:"price_currency"`
    Description   string    `json:"description"`
    Category      string    `json:"category"`
    Stock         int       `json:"stock"`
    CreatedAt     time.Time `json:"created_at"`
}

type Money struct {
    Amount   float64 `json:"amount"`
    Currency string  `json:"currency"`
}

type OrderV2 struct {
    ID        int       `json:"id"`
    Product   ProductV2 `json:"product"`
    Quantity  int       `json:"quantity"`
    Total     Money     `json:"total"`
    CreatedAt time.Time `json:"created_at"`
}

// =============================================================================
// DATABASE MODELS
// =============================================================================

type ProductRecord struct {
    ID            int
    Name          string
    PriceAmount   float64
    PriceCurrency string
    Description   string
    Category      string
    Stock         int
    CreatedAt     time.Time
}

type OrderRecord struct {
    ID          int
    ProductID   int
    Quantity    int
    TotalAmount float64
    Currency    string
    CreatedAt   time.Time
}

// Storage
var (
    products    = make(map[int]ProductRecord)
    orders      = make(map[int]OrderRecord)
    nextProdID  = 1
    nextOrderID = 1
    productsMu  sync.RWMutex
    ordersMu    sync.RWMutex
)

// TODO: Implement V1 product handlers
func getProductsV1(w http.ResponseWriter, r *http.Request) {
    // Hint: Get ProductRecords, convert to ProductV1
}

func getProductV1(w http.ResponseWriter, r *http.Request) {
    // Hint: Get single ProductRecord, convert to ProductV1
}

// TODO: Implement V2 product handlers
func getProductsV2(w http.ResponseWriter, r *http.Request) {
    // Hint: Get ProductRecords, convert to ProductV2
}

func getProductV2(w http.ResponseWriter, r *http.Request) {
    // Hint: Get single ProductRecord, convert to ProductV2
}

// TODO: Implement V1 order handlers
func createOrderV1(w http.ResponseWriter, r *http.Request) {
    // Hint: Parse OrderV1, create OrderRecord, return OrderV1
}

func getOrderV1(w http.ResponseWriter, r *http.Request) {
    // Hint: Get OrderRecord, convert to OrderV1
}

// TODO: Implement V2 order handlers
func createOrderV2(w http.ResponseWriter, r *http.Request) {
    // Hint: Parse OrderV2, create OrderRecord, return OrderV2
}

func getOrderV2(w http.ResponseWriter, r *http.Request) {
    // Hint: Get OrderRecord, convert to OrderV2 with full product
}

// TODO: Implement conversion functions
func productRecordToV1(record ProductRecord) ProductV1 {
    // Convert ProductRecord to ProductV1
}

func productRecordToV2(record ProductRecord) ProductV2 {
    // Convert ProductRecord to ProductV2
}

func orderRecordToV1(orderRec OrderRecord, prodRec ProductRecord) OrderV1 {
    // Convert to OrderV1
}

func orderRecordToV2(orderRec OrderRecord, prodRec ProductRecord) OrderV2 {
    // Convert to OrderV2 with embedded product
}

// TODO: Implement deprecation middleware
func deprecationMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Add deprecation headers
    })
}

// Helpers
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}

func main() {
    // TODO: Seed database with sample products
    
    r := mux.NewRouter()
    
    // TODO: Register V1 routes with deprecation middleware
    // TODO: Register V2 routes
    
    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", r)
}
```

---

## Sample Data to Seed

```go
func seedDatabase() {
    productsMu.Lock()
    defer productsMu.Unlock()
    
    products[1] = ProductRecord{
        ID:            1,
        Name:          "Laptop",
        PriceAmount:   999.99,
        PriceCurrency: "USD",
        Description:   "High-performance laptop",
        Category:      "Electronics",
        Stock:         10,
        CreatedAt:     time.Now(),
    }
    
    products[2] = ProductRecord{
        ID:            2,
        Name:          "Mouse",
        PriceAmount:   29.99,
        PriceCurrency: "USD",
        Description:   "Wireless mouse",
        Category:      "Accessories",
        Stock:         50,
        CreatedAt:     time.Now(),
    }
    
    products[3] = ProductRecord{
        ID:            3,
        Name:          "Keyboard",
        PriceAmount:   79.99,
        PriceCurrency: "USD",
        Description:   "Mechanical keyboard",
        Category:      "Accessories",
        Stock:         25,
        CreatedAt:     time.Now(),
    }
    
    nextProdID = 4
}
```

---

## Testing Your API

### Test V1 Endpoints

```bash
# Get all products (V1)
curl http://localhost:8080/api/v1/products

# Expected response:
# [
#   {
#     "id": 1,
#     "name": "Laptop",
#     "price": 999.99,
#     "description": "High-performance laptop"
#   }
# ]

# Get single product (V1)
curl http://localhost:8080/api/v1/products/1

# Create order (V1)
curl -X POST http://localhost:8080/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{
    "product_id": 1,
    "quantity": 2
  }'

# Expected response:
# {
#   "id": 1,
#   "product_id": 1,
#   "quantity": 2,
#   "total_price": 1999.98
# }
```

### Test V2 Endpoints

```bash
# Get all products (V2)
curl http://localhost:8080/api/v2/products

# Expected response:
# [
#   {
#     "id": 1,
#     "name": "Laptop",
#     "price_amount": 999.99,
#     "price_currency": "USD",
#     "description": "High-performance laptop",
#     "category": "Electronics",
#     "stock": 10,
#     "created_at": "2024-01-15T10:30:00Z"
#   }
# ]

# Get single product (V2)
curl http://localhost:8080/api/v2/products/1

# Create order (V2)
curl -X POST http://localhost:8080/api/v2/orders \
  -H "Content-Type: application/json" \
  -d '{
    "product_id": 1,
    "quantity": 2
  }'

# Expected response:
# {
#   "id": 1,
#   "product": {
#     "id": 1,
#     "name": "Laptop",
#     "price_amount": 999.99,
#     "price_currency": "USD",
#     ...
#   },
#   "quantity": 2,
#   "total": {
#     "amount": 1999.98,
#     "currency": "USD"
#   },
#   "created_at": "2024-01-15T11:00:00Z"
# }
```

### Test Deprecation Headers

```bash
# V1 should include deprecation headers
curl -i http://localhost:8080/api/v1/products

# Look for headers:
# Deprecation: true
# Sunset: 2024-12-31
# Link: </api/v2/products>; rel="successor-version"
```

---

## Validation Requirements

### V1 Create Order
- Product ID must exist
- Quantity must be positive
- Calculate total_price automatically

### V2 Create Order
- Product ID must exist
- Quantity must be positive
- Check if enough stock available
- Calculate total automatically
- Set created_at timestamp

---

## Expected Behavior

### Price Handling
- **V1**: Single price field in USD
- **V2**: Separate amount and currency fields

### Order Response
- **V1**: Only product_id
- **V2**: Full product object embedded

### Timestamps
- **V1**: No timestamps
- **V2**: created_at on products and orders

---

## Bonus Challenges

### Bonus 1: Multi-Currency Support
Add currency conversion for V2:
- Store products in different currencies
- Convert to requested currency
- Add `?currency=EUR` query parameter

### Bonus 2: Stock Management
Implement stock tracking:
- Decrease stock when order created
- Return error if insufficient stock
- Add stock update endpoint (V2 only)

### Bonus 3: Search and Filter
Add filtering to V2:
```bash
GET /api/v2/products?category=Electronics
GET /api/v2/products?min_price=50&max_price=100
```

### Bonus 4: Batch Operations
Add batch endpoints to V2:
```bash
POST /api/v2/orders/batch
# Create multiple orders at once
```

### Bonus 5: Migration Endpoint
Create a migration helper:
```bash
GET /api/v1/orders/1?format=v2
# Returns V1 order in V2 format for testing
```

---

## Hints

### Hint 1: Product Conversion

```go
func productRecordToV1(record ProductRecord) ProductV1 {
    return ProductV1{
        ID:          record.ID,
        Name:        record.Name,
        Price:       record.PriceAmount,  // V1 doesn't show currency
        Description: record.Description,
    }
}

func productRecordToV2(record ProductRecord) ProductV2 {
    return ProductV2{
        ID:            record.ID,
        Name:          record.Name,
        PriceAmount:   record.PriceAmount,
        PriceCurrency: record.PriceCurrency,
        Description:   record.Description,
        Category:      record.Category,
        Stock:         record.Stock,
        CreatedAt:     record.CreatedAt,
    }
}
```

### Hint 2: Create Order V1

```go
func createOrderV1(w http.ResponseWriter, r *http.Request) {
    var req struct {
        ProductID int `json:"product_id"`
        Quantity  int `json:"quantity"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }

    // Get product
    productsMu.RLock()
    product, exists := products[req.ProductID]
    productsMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Product not found")
        return
    }

    // Calculate total
    total := product.PriceAmount * float64(req.Quantity)

    // Create order record
    ordersMu.Lock()
    order := OrderRecord{
        ID:          nextOrderID,
        ProductID:   req.ProductID,
        Quantity:    req.Quantity,
        TotalAmount: total,
        Currency:    product.PriceCurrency,
        CreatedAt:   time.Now(),
    }
    orders[nextOrderID] = order
    nextOrderID++
    ordersMu.Unlock()

    // Return V1 format
    v1Order := OrderV1{
        ID:         order.ID,
        ProductID:  order.ProductID,
        Quantity:   order.Quantity,
        TotalPrice: order.TotalAmount,
    }

    respondJSON(w, http.StatusCreated, v1Order)
}
```

### Hint 3: Deprecation Middleware

```go
func deprecationMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Deprecation", "true")
        w.Header().Set("Sunset", "2024-12-31")
        w.Header().Set("Link", fmt.Sprintf("<%s>; rel=\"successor-version\"", 
            strings.Replace(r.URL.Path, "/v1/", "/v2/", 1)))
        next.ServeHTTP(w, r)
    })
}
```

---

## What You're Learning

✅ Maintaining multiple API versions simultaneously  
✅ Converting between different data formats  
✅ Sharing business logic across versions  
✅ Adding deprecation headers  
✅ Handling breaking changes gracefully  
✅ Database model separate from API models  
✅ Backward compatibility patterns  

This exercise demonstrates real-world API evolution!
