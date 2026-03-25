# Unit 5 - Exercise 1: Document a Product Catalog API

**Difficulty**: Intermediate  
**Estimated Time**: 45-60 minutes  
**Concepts Covered**: Swagger annotations, model documentation, interactive API testing

---

## Objective

Take an existing Product Catalog API and add complete Swagger/OpenAPI documentation. Learn how to document CRUD operations, query parameters, authentication, and error responses.

---

## Requirements

### API to Document

A product catalog API with the following endpoints:

| Method | Path | Description | Auth Required |
|--------|------|-------------|---------------|
| GET | /products | List all products (with filters) | No |
| GET | /products/{id} | Get single product | No |
| POST | /products | Create product | Yes |
| PUT | /products/{id} | Update product | Yes |
| DELETE | /products/{id} | Delete product | Yes |
| POST | /products/{id}/review | Add review | Yes |
| GET | /categories | List categories | No |

### Models to Document

```go
type Product struct {
    ID          int       `json:"id"`
    Name        string    `json:"name"`
    Description string    `json:"description"`
    Price       float64   `json:"price"`
    Category    string    `json:"category"`
    Stock       int       `json:"stock"`
    ImageURL    string    `json:"image_url"`
    Rating      float64   `json:"rating"`
    CreatedAt   time.Time `json:"created_at"`
}

type CreateProductRequest struct {
    Name        string  `json:"name"`
    Description string  `json:"description"`
    Price       float64 `json:"price"`
    Category    string  `json:"category"`
    Stock       int     `json:"stock"`
    ImageURL    string  `json:"image_url"`
}

type Review struct {
    ID        int       `json:"id"`
    ProductID int       `json:"product_id"`
    UserID    int       `json:"user_id"`
    Rating    int       `json:"rating"`
    Comment   string    `json:"comment"`
    CreatedAt time.Time `json:"created_at"`
}

type CreateReviewRequest struct {
    Rating  int    `json:"rating"`
    Comment string `json:"comment"`
}
```

### Query Parameters for GET /products

- `category` (string): Filter by category
- `min_price` (float): Minimum price
- `max_price` (float): Maximum price
- `in_stock` (bool): Only in-stock items
- `sort` (string): Sort by (name, price, rating)
- `page` (int): Page number (default: 1)
- `limit` (int): Items per page (default: 10)

---

## Starter Code

```go
package main

import (
    "encoding/json"
    "net/http"
    "strconv"
    "sync"
    "time"

    "github.com/gorilla/mux"
    httpSwagger "github.com/swaggo/http-swagger"

    _ "myapi/docs"
)

// TODO: Add general API information annotations
// @title
// @version
// @description
// @host
// @BasePath
// @securityDefinitions.apikey BearerAuth

type Product struct {
    ID          int       `json:"id"`
    Name        string    `json:"name"`
    Description string    `json:"description"`
    Price       float64   `json:"price"`
    Category    string    `json:"category"`
    Stock       int       `json:"stock"`
    ImageURL    string    `json:"image_url"`
    Rating      float64   `json:"rating"`
    CreatedAt   time.Time `json:"created_at"`
}

type CreateProductRequest struct {
    Name        string  `json:"name" binding:"required"`
    Description string  `json:"description"`
    Price       float64 `json:"price" binding:"required"`
    Category    string  `json:"category" binding:"required"`
    Stock       int     `json:"stock" binding:"required"`
    ImageURL    string  `json:"image_url"`
}

type Review struct {
    ID        int       `json:"id"`
    ProductID int       `json:"product_id"`
    UserID    int       `json:"user_id"`
    Rating    int       `json:"rating"`
    Comment   string    `json:"comment"`
    CreatedAt time.Time `json:"created_at"`
}

type CreateReviewRequest struct {
    Rating  int    `json:"rating" binding:"required"`
    Comment string `json:"comment"`
}

type ErrorResponse struct {
    Error string `json:"error"`
}

// Storage
var (
    products    = make(map[int]Product)
    reviews     = make(map[int]Review)
    nextProdID  = 1
    nextRevID   = 1
    productsMu  sync.RWMutex
    reviewsMu   sync.RWMutex
)

// TODO: Document this handler
func getProducts(w http.ResponseWriter, r *http.Request) {
    // Parse query parameters
    category := r.URL.Query().Get("category")
    minPrice := r.URL.Query().Get("min_price")
    maxPrice := r.URL.Query().Get("max_price")
    inStock := r.URL.Query().Get("in_stock")

    productsMu.RLock()
    defer productsMu.RUnlock()

    results := []Product{}
    for _, p := range products {
        // Apply filters
        if category != "" && p.Category != category {
            continue
        }

        if minPrice != "" {
            min, _ := strconv.ParseFloat(minPrice, 64)
            if p.Price < min {
                continue
            }
        }

        if maxPrice != "" {
            max, _ := strconv.ParseFloat(maxPrice, 64)
            if p.Price > max {
                continue
            }
        }

        if inStock == "true" && p.Stock <= 0 {
            continue
        }

        results = append(results, p)
    }

    respondJSON(w, http.StatusOK, results)
}

// TODO: Document this handler
func getProduct(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    productsMu.RLock()
    product, exists := products[id]
    productsMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Product not found")
        return
    }

    respondJSON(w, http.StatusOK, product)
}

// TODO: Document this handler (requires auth)
func createProduct(w http.ResponseWriter, r *http.Request) {
    var req CreateProductRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }

    productsMu.Lock()
    product := Product{
        ID:          nextProdID,
        Name:        req.Name,
        Description: req.Description,
        Price:       req.Price,
        Category:    req.Category,
        Stock:       req.Stock,
        ImageURL:    req.ImageURL,
        Rating:      0.0,
        CreatedAt:   time.Now(),
    }
    products[nextProdID] = product
    nextProdID++
    productsMu.Unlock()

    respondJSON(w, http.StatusCreated, product)
}

// TODO: Document this handler (requires auth)
func updateProduct(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    var req CreateProductRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }

    productsMu.Lock()
    product, exists := products[id]
    if !exists {
        productsMu.Unlock()
        respondError(w, http.StatusNotFound, "Product not found")
        return
    }

    product.Name = req.Name
    product.Description = req.Description
    product.Price = req.Price
    product.Category = req.Category
    product.Stock = req.Stock
    product.ImageURL = req.ImageURL
    products[id] = product
    productsMu.Unlock()

    respondJSON(w, http.StatusOK, product)
}

// TODO: Document this handler (requires auth)
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

    w.WriteHeader(http.StatusNoContent)
}

// TODO: Document this handler (requires auth)
func createReview(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    productID, _ := strconv.Atoi(vars["id"])

    // Check product exists
    productsMu.RLock()
    _, exists := products[productID]
    productsMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Product not found")
        return
    }

    var req CreateReviewRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }

    // Validate rating
    if req.Rating < 1 || req.Rating > 5 {
        respondError(w, http.StatusBadRequest, "Rating must be between 1 and 5")
        return
    }

    reviewsMu.Lock()
    review := Review{
        ID:        nextRevID,
        ProductID: productID,
        UserID:    1, // Would come from auth token
        Rating:    req.Rating,
        Comment:   req.Comment,
        CreatedAt: time.Now(),
    }
    reviews[nextRevID] = review
    nextRevID++
    reviewsMu.Unlock()

    // Update product rating (simplified)
    updateProductRating(productID)

    respondJSON(w, http.StatusCreated, review)
}

// TODO: Document this handler
func getCategories(w http.ResponseWriter, r *http.Request) {
    productsMu.RLock()
    defer productsMu.RUnlock()

    categorySet := make(map[string]bool)
    for _, p := range products {
        categorySet[p.Category] = true
    }

    categories := []string{}
    for cat := range categorySet {
        categories = append(categories, cat)
    }

    respondJSON(w, http.StatusOK, categories)
}

func updateProductRating(productID int) {
    reviewsMu.RLock()
    defer reviewsMu.RUnlock()

    totalRating := 0
    count := 0
    for _, r := range reviews {
        if r.ProductID == productID {
            totalRating += r.Rating
            count++
        }
    }

    if count > 0 {
        productsMu.Lock()
        product := products[productID]
        product.Rating = float64(totalRating) / float64(count)
        products[productID] = product
        productsMu.Unlock()
    }
}

func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, ErrorResponse{Error: message})
}

func seedDatabase() {
    productsMu.Lock()
    defer productsMu.Unlock()

    products[1] = Product{
        ID:          1,
        Name:        "Laptop",
        Description: "High-performance laptop",
        Price:       999.99,
        Category:    "Electronics",
        Stock:       10,
        ImageURL:    "https://example.com/laptop.jpg",
        Rating:      4.5,
        CreatedAt:   time.Now(),
    }

    products[2] = Product{
        ID:          2,
        Name:        "Coffee Mug",
        Description: "Ceramic coffee mug",
        Price:       12.99,
        Category:    "Home",
        Stock:       50,
        ImageURL:    "https://example.com/mug.jpg",
        Rating:      4.0,
        CreatedAt:   time.Now(),
    }

    nextProdID = 3
}

func main() {
    seedDatabase()

    r := mux.NewRouter()

    // API routes
    api := r.PathPrefix("/api/v1").Subrouter()
    api.HandleFunc("/products", getProducts).Methods("GET")
    api.HandleFunc("/products/{id}", getProduct).Methods("GET")
    api.HandleFunc("/products", createProduct).Methods("POST")
    api.HandleFunc("/products/{id}", updateProduct).Methods("PUT")
    api.HandleFunc("/products/{id}", deleteProduct).Methods("DELETE")
    api.HandleFunc("/products/{id}/review", createReview).Methods("POST")
    api.HandleFunc("/categories", getCategories).Methods("GET")

    // Swagger UI
    r.PathPrefix("/swagger/").Handler(httpSwagger.WrapHandler)

    http.ListenAndServe(":8080", r)
}
```

---

## Your Tasks

### Task 1: Add General API Info

Add these annotations above `func main()`:
- Title: "Product Catalog API"
- Version: "1.0"
- Description: "A product catalog with categories and reviews"
- Host: "localhost:8080"
- BasePath: "/api/v1"
- Security: BearerAuth (API key in header)

### Task 2: Document All Models

Add `example` tags to all struct fields:
- Product fields
- CreateProductRequest fields
- Review fields
- CreateReviewRequest fields
- ErrorResponse field

### Task 3: Document Each Handler

For each handler, add:
- `@Summary`: One-line description
- `@Description`: Detailed description
- `@Tags`: Group by "products", "reviews", or "categories"
- `@Accept`: json
- `@Produce`: json
- `@Param`: All parameters (path, query, body)
- `@Success`: Success response with model
- `@Failure`: Error responses (400, 404, 500)
- `@Security`: BearerAuth (for protected endpoints)
- `@Router`: Path and method

### Task 4: Document Query Parameters

For `getProducts`, document:
- category (string, optional)
- min_price (number, optional)
- max_price (number, optional)
- in_stock (boolean, optional)
- sort (string, optional) - enum: "name,price,rating"
- page (integer, optional, default: 1)
- limit (integer, optional, default: 10)

### Task 5: Generate and Test

1. Run `swag init`
2. Start the server
3. Visit `http://localhost:8080/swagger/index.html`
4. Test each endpoint in Swagger UI
5. Try the "Authorize" button with a sample token

---

## Expected Documentation

### Example: GET /products

```go
// @Summary      List products
// @Description  Get a list of products with optional filters
// @Tags         products
// @Accept       json
// @Produce      json
// @Param        category   query     string  false  "Filter by category"
// @Param        min_price  query     number  false  "Minimum price"
// @Param        max_price  query     number  false  "Maximum price"
// @Param        in_stock   query     boolean false  "Only in-stock items"
// @Param        sort       query     string  false  "Sort by" Enums(name, price, rating)
// @Param        page       query     int     false  "Page number" default(1)
// @Param        limit      query     int     false  "Items per page" default(10)
// @Success      200        {array}   Product
// @Failure      500        {object}  ErrorResponse
// @Router       /products [get]
func getProducts(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

### Example: POST /products

```go
// @Summary      Create product
// @Description  Create a new product (admin only)
// @Tags         products
// @Accept       json
// @Produce      json
// @Param        product  body      CreateProductRequest  true  "Product data"
// @Success      201      {object}  Product
// @Failure      400      {object}  ErrorResponse
// @Security     BearerAuth
// @Router       /products [post]
func createProduct(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

---

## Testing Checklist

- [ ] Swagger UI loads at `/swagger/index.html`
- [ ] All endpoints are visible
- [ ] Grouped by tags (products, reviews, categories)
- [ ] Models show example values
- [ ] Query parameters have descriptions
- [ ] Required fields are marked
- [ ] Auth button works
- [ ] Can test endpoints directly
- [ ] Error responses documented
- [ ] All HTTP methods shown correctly

---

## Bonus Challenges

### Bonus 1: Add Pagination Response
Document pagination metadata:
```go
type PaginatedProducts struct {
    Data       []Product `json:"data"`
    Page       int       `json:"page"`
    Limit      int       `json:"limit"`
    TotalItems int       `json:"total_items"`
    TotalPages int       `json:"total_pages"`
}
```

### Bonus 2: Add Search Endpoint
```go
// GET /products/search?q=laptop
```

### Bonus 3: Document Response Headers
Add deprecation warnings:
```go
// @Header 200 {string} X-Rate-Limit "Requests per hour"
// @Header 200 {string} X-RateLimit-Remaining "Remaining requests"
```

### Bonus 4: Add Examples
Include request/response examples:
```go
// @Success 200 {object} Product "Product retrieved successfully"
// @Success 200 {object} Product{name=string,price=number} "Custom example"
```

### Bonus 5: Multiple Security Schemes
Add API key support:
```go
// @securityDefinitions.apikey ApiKeyAuth
// @in header
// @name X-API-Key

// Then use:
// @Security BearerAuth || ApiKeyAuth
```

---

## Hints

### Hint 1: Model Example Tags

```go
type Product struct {
    ID          int       `json:"id" example:"1"`
    Name        string    `json:"name" example:"Laptop"`
    Price       float64   `json:"price" example:"999.99"`
    Category    string    `json:"category" example:"Electronics"`
    Stock       int       `json:"stock" example:"10"`
    Rating      float64   `json:"rating" example:"4.5"`
}
```

### Hint 2: Query Parameter Enums

```go
// @Param sort query string false "Sort by" Enums(name, price, rating)
```

### Hint 3: Path Parameters

```go
// @Param id path int true "Product ID"
```

### Hint 4: Body Parameters

```go
// @Param product body CreateProductRequest true "Product data"
```

---

## What You're Learning

✅ Writing Swagger annotations  
✅ Documenting query parameters  
✅ Adding example values  
✅ Grouping endpoints with tags  
✅ Documenting authentication  
✅ Testing APIs interactively  
✅ Generating OpenAPI specs  
✅ Best practices for API docs  

This creates professional, interactive API documentation!
