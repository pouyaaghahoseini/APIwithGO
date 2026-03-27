# Unit 4 - Exercise 1 Solution: E-commerce API Migration (v1 to v2)

**Complete implementation with explanations**

---

## Full Solution Code

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strconv"
    "strings"
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

// =============================================================================
// STORAGE
// =============================================================================

var (
    products    = make(map[int]ProductRecord)
    orders      = make(map[int]OrderRecord)
    nextProdID  = 1
    nextOrderID = 1
    productsMu  sync.RWMutex
    ordersMu    sync.RWMutex
)

// =============================================================================
// V1 HANDLERS
// =============================================================================

func getProductsV1(w http.ResponseWriter, r *http.Request) {
    productsMu.RLock()
    defer productsMu.RUnlock()

    v1Products := make([]ProductV1, 0, len(products))
    for _, record := range products {
        v1Products = append(v1Products, productRecordToV1(record))
    }

    respondJSON(w, http.StatusOK, v1Products)
}

func getProductV1(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    productsMu.RLock()
    record, exists := products[id]
    productsMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Product not found")
        return
    }

    respondJSON(w, http.StatusOK, productRecordToV1(record))
}

func createOrderV1(w http.ResponseWriter, r *http.Request) {
    var req struct {
        ProductID int `json:"product_id"`
        Quantity  int `json:"quantity"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Validate
    if req.Quantity <= 0 {
        respondError(w, http.StatusBadRequest, "Quantity must be positive")
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

    // Create order
    now := time.Now()
    ordersMu.Lock()
    order := OrderRecord{
        ID:          nextOrderID,
        ProductID:   req.ProductID,
        Quantity:    req.Quantity,
        TotalAmount: total,
        Currency:    product.PriceCurrency,
        CreatedAt:   now,
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

func getOrderV1(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    ordersMu.RLock()
    orderRec, exists := orders[id]
    ordersMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Order not found")
        return
    }

    productsMu.RLock()
    productRec := products[orderRec.ProductID]
    productsMu.RUnlock()

    v1Order := orderRecordToV1(orderRec, productRec)
    respondJSON(w, http.StatusOK, v1Order)
}

// =============================================================================
// V2 HANDLERS
// =============================================================================

func getProductsV2(w http.ResponseWriter, r *http.Request) {
    productsMu.RLock()
    defer productsMu.RUnlock()

    v2Products := make([]ProductV2, 0, len(products))
    for _, record := range products {
        v2Products = append(v2Products, productRecordToV2(record))
    }

    respondJSON(w, http.StatusOK, v2Products)
}

func getProductV2(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    productsMu.RLock()
    record, exists := products[id]
    productsMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Product not found")
        return
    }

    respondJSON(w, http.StatusOK, productRecordToV2(record))
}

func createOrderV2(w http.ResponseWriter, r *http.Request) {
    var req struct {
        ProductID int `json:"product_id"`
        Quantity  int `json:"quantity"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Validate
    if req.Quantity <= 0 {
        respondError(w, http.StatusBadRequest, "Quantity must be positive")
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

    // Check stock (V2 feature)
    if product.Stock < req.Quantity {
        respondError(w, http.StatusBadRequest, 
            fmt.Sprintf("Insufficient stock. Available: %d", product.Stock))
        return
    }

    // Calculate total
    total := product.PriceAmount * float64(req.Quantity)

    // Create order
    now := time.Now()
    ordersMu.Lock()
    order := OrderRecord{
        ID:          nextOrderID,
        ProductID:   req.ProductID,
        Quantity:    req.Quantity,
        TotalAmount: total,
        Currency:    product.PriceCurrency,
        CreatedAt:   now,
    }
    orders[nextOrderID] = order
    nextOrderID++
    ordersMu.Unlock()

    // Update stock
    productsMu.Lock()
    product.Stock -= req.Quantity
    products[req.ProductID] = product
    productsMu.Unlock()

    // Return V2 format (with full product details)
    v2Order := orderRecordToV2(order, product)
    respondJSON(w, http.StatusCreated, v2Order)
}

func getOrderV2(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    ordersMu.RLock()
    orderRec, exists := orders[id]
    ordersMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Order not found")
        return
    }

    productsMu.RLock()
    productRec := products[orderRec.ProductID]
    productsMu.RUnlock()

    v2Order := orderRecordToV2(orderRec, productRec)
    respondJSON(w, http.StatusOK, v2Order)
}

// =============================================================================
// CONVERSION FUNCTIONS
// =============================================================================

func productRecordToV1(record ProductRecord) ProductV1 {
    return ProductV1{
        ID:          record.ID,
        Name:        record.Name,
        Price:       record.PriceAmount, // V1 doesn't show currency
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

func orderRecordToV1(orderRec OrderRecord, productRec ProductRecord) OrderV1 {
    return OrderV1{
        ID:         orderRec.ID,
        ProductID:  orderRec.ProductID,
        Quantity:   orderRec.Quantity,
        TotalPrice: orderRec.TotalAmount,
    }
}

func orderRecordToV2(orderRec OrderRecord, productRec ProductRecord) OrderV2 {
    return OrderV2{
        ID:       orderRec.ID,
        Product:  productRecordToV2(productRec),
        Quantity: orderRec.Quantity,
        Total: Money{
            Amount:   orderRec.TotalAmount,
            Currency: orderRec.Currency,
        },
        CreatedAt: orderRec.CreatedAt,
    }
}

// =============================================================================
// MIDDLEWARE
// =============================================================================

func deprecationMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Add deprecation headers
        w.Header().Set("Deprecation", "true")
        w.Header().Set("Sunset", "2024-12-31")
        
        // Suggest V2 endpoint
        v2Path := strings.Replace(r.URL.Path, "/v1/", "/v2/", 1)
        w.Header().Set("Link", fmt.Sprintf("<%s>; rel=\"successor-version\"", v2Path))
        w.Header().Set("X-Migration-Guide", "https://docs.api.com/migration/v1-to-v2")
        
        next.ServeHTTP(w, r)
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
    productsMu.Lock()
    defer productsMu.Unlock()

    now := time.Now()

    products[1] = ProductRecord{
        ID:            1,
        Name:          "Laptop",
        PriceAmount:   999.99,
        PriceCurrency: "USD",
        Description:   "High-performance laptop",
        Category:      "Electronics",
        Stock:         10,
        CreatedAt:     now,
    }

    products[2] = ProductRecord{
        ID:            2,
        Name:          "Mouse",
        PriceAmount:   29.99,
        PriceCurrency: "USD",
        Description:   "Wireless mouse",
        Category:      "Accessories",
        Stock:         50,
        CreatedAt:     now,
    }

    products[3] = ProductRecord{
        ID:            3,
        Name:          "Keyboard",
        PriceAmount:   79.99,
        PriceCurrency: "USD",
        Description:   "Mechanical keyboard",
        Category:      "Accessories",
        Stock:         25,
        CreatedAt:     now,
    }

    nextProdID = 4
}

// =============================================================================
// MAIN
// =============================================================================

func main() {
    seedDatabase()

    r := mux.NewRouter()

    // V1 routes (deprecated)
    v1 := r.PathPrefix("/api/v1").Subrouter()
    v1.Use(deprecationMiddleware)
    v1.HandleFunc("/products", getProductsV1).Methods("GET")
    v1.HandleFunc("/products/{id}", getProductV1).Methods("GET")
    v1.HandleFunc("/orders", createOrderV1).Methods("POST")
    v1.HandleFunc("/orders/{id}", getOrderV1).Methods("GET")

    // V2 routes (current)
    v2 := r.PathPrefix("/api/v2").Subrouter()
    v2.HandleFunc("/products", getProductsV2).Methods("GET")
    v2.HandleFunc("/products/{id}", getProductV2).Methods("GET")
    v2.HandleFunc("/orders", createOrderV2).Methods("POST")
    v2.HandleFunc("/orders/{id}", getOrderV2).Methods("GET")

    fmt.Println("Server starting on :8080")
    fmt.Println("V1 API (deprecated): http://localhost:8080/api/v1")
    fmt.Println("V2 API (current):    http://localhost:8080/api/v2")
    http.ListenAndServe(":8080", r)
}
```

---

## Key Concepts Explained

### 1. Shared Database Model

```go
// Single source of truth
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
```

**Why?** One database representation supports all API versions. This avoids data duplication and migration complexity.

### 2. Model Conversion Functions

```go
// V1 conversion - hides complexity
func productRecordToV1(record ProductRecord) ProductV1 {
    return ProductV1{
        ID:          record.ID,
        Name:        record.Name,
        Price:       record.PriceAmount,  // Currency hidden
        Description: record.Description,
    }
}

// V2 conversion - exposes everything
func productRecordToV2(record ProductRecord) ProductV2 {
    return ProductV2{
        ID:            record.ID,
        Name:          record.Name,
        PriceAmount:   record.PriceAmount,
        PriceCurrency: record.PriceCurrency,  // Explicit currency
        Description:   record.Description,
        Category:      record.Category,
        Stock:         record.Stock,
        CreatedAt:     record.CreatedAt,
    }
}
```

**Benefits**:
- Centralized conversion logic
- Easy to add new versions
- Clear mapping between formats

### 3. Deprecation Headers

```go
func deprecationMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Deprecation", "true")
        w.Header().Set("Sunset", "2024-12-31")
        
        // Dynamic successor link
        v2Path := strings.Replace(r.URL.Path, "/v1/", "/v2/", 1)
        w.Header().Set("Link", fmt.Sprintf("<%s>; rel=\"successor-version\"", v2Path))
        
        next.ServeHTTP(w, r)
    })
}
```

**Headers sent**:
```
Deprecation: true
Sunset: 2024-12-31
Link: </api/v2/products>; rel="successor-version"
X-Migration-Guide: https://docs.api.com/migration/v1-to-v2
```

### 4. Version-Specific Features

```go
// V1: No stock checking
func createOrderV1(w http.ResponseWriter, r *http.Request) {
    // ... just creates order
}

// V2: Checks and updates stock
func createOrderV2(w http.ResponseWriter, r *http.Request) {
    if product.Stock < req.Quantity {
        respondError(w, 400, "Insufficient stock")
        return
    }
    
    // ... creates order AND updates stock
    product.Stock -= req.Quantity
}
```

### 5. Response Format Differences

**V1 Order Response**:
```json
{
  "id": 1,
  "product_id": 1,
  "quantity": 2,
  "total_price": 1999.98
}
```

**V2 Order Response**:
```json
{
  "id": 1,
  "product": {
    "id": 1,
    "name": "Laptop",
    "price_amount": 999.99,
    "price_currency": "USD",
    "category": "Electronics",
    "stock": 8,
    "created_at": "2024-01-15T10:30:00Z"
  },
  "quantity": 2,
  "total": {
    "amount": 1999.98,
    "currency": "USD"
  },
  "created_at": "2024-01-15T11:00:00Z"
}
```

---

## Testing the Solution

### Test Script

```bash
#!/bin/bash

BASE="http://localhost:8080"

echo "=== Testing V1 (Deprecated) ==="
echo "Get products (V1):"
curl -i $BASE/api/v1/products
echo -e "\n"

echo "Get single product (V1):"
curl $BASE/api/v1/products/1 | jq .
echo -e "\n"

echo "Create order (V1):"
curl -X POST $BASE/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{"product_id":2,"quantity":3}' | jq .
echo -e "\n"

echo "=== Testing V2 (Current) ==="
echo "Get products (V2):"
curl $BASE/api/v2/products | jq .
echo -e "\n"

echo "Get single product (V2):"
curl $BASE/api/v2/products/1 | jq .
echo -e "\n"

echo "Create order (V2):"
curl -X POST $BASE/api/v2/orders \
  -H "Content-Type: application/json" \
  -d '{"product_id":1,"quantity":2}' | jq .
echo -e "\n"

echo "Get order (V2 - with embedded product):"
curl $BASE/api/v2/orders/2 | jq .
echo -e "\n"

echo "Test stock validation (V2):"
curl -X POST $BASE/api/v2/orders \
  -H "Content-Type: application/json" \
  -d '{"product_id":1,"quantity":100}' | jq .
echo -e "\n"
```

---

## Comparison: V1 vs V2

| Feature | V1 | V2 |
|---------|----|----|
| Price | Single field | Amount + Currency |
| Product Info | ID, Name, Price | + Category, Stock, CreatedAt |
| Order Response | Product ID only | Full product object |
| Stock Check | No | Yes |
| Stock Update | No | Yes |
| Timestamps | No | Yes on orders |
| Deprecation | Headers present | None |

---

## Migration Path

### For Clients Moving from V1 to V2

**Changes Required**:

1. **Product price** - Change from `price` to `price_amount` + `price_currency`
   ```javascript
   // V1
   const total = product.price * quantity;
   
   // V2
   const total = product.price_amount * quantity;
   const currency = product.price_currency;
   ```

2. **Order response** - Extract product from embedded object
   ```javascript
   // V1
   const productId = order.product_id;
   
   // V2
   const product = order.product;
   const productId = product.id;
   ```

3. **Handle stock errors** - V2 returns 400 if insufficient stock
   ```javascript
   // V2 only
   if (response.status === 400) {
     console.log("Out of stock!");
   }
   ```

---

## What You've Learned

✅ **URL path versioning** implementation  
✅ **Shared database model** supporting multiple versions  
✅ **Model conversion** between API versions  
✅ **Deprecation headers** to guide migration  
✅ **Version-specific features** (stock checking in V2)  
✅ **Response format changes** (embedded objects)  
✅ **Backward compatibility** - both versions work simultaneously  

This demonstrates real-world API evolution patterns!
