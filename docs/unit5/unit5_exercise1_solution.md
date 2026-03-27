# Unit 5 - Exercise 1 Solution: Document a Product Catalog API

**Complete implementation with Swagger documentation**

---

## Full Solution Code

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

// @title           Product Catalog API
// @version         1.0
// @description     A product catalog API with categories, reviews, and filtering capabilities
// @termsOfService  http://swagger.io/terms/

// @contact.name   API Support
// @contact.url    http://www.example.com/support
// @contact.email  support@example.com

// @license.name  MIT
// @license.url   https://opensource.org/licenses/MIT

// @host      localhost:8080
// @BasePath  /api/v1

// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization
// @description Type "Bearer" followed by a space and JWT token.

// Product represents a product in the catalog
type Product struct {
    ID          int       `json:"id" example:"1"`
    Name        string    `json:"name" example:"Laptop"`
    Description string    `json:"description" example:"High-performance laptop"`
    Price       float64   `json:"price" example:"999.99"`
    Category    string    `json:"category" example:"Electronics"`
    Stock       int       `json:"stock" example:"10"`
    ImageURL    string    `json:"image_url" example:"https://example.com/laptop.jpg"`
    Rating      float64   `json:"rating" example:"4.5"`
    CreatedAt   time.Time `json:"created_at" example:"2024-01-15T10:30:00Z"`
}

// CreateProductRequest is the request body for creating a product
type CreateProductRequest struct {
    Name        string  `json:"name" example:"Laptop" binding:"required"`
    Description string  `json:"description" example:"High-performance laptop"`
    Price       float64 `json:"price" example:"999.99" binding:"required"`
    Category    string  `json:"category" example:"Electronics" binding:"required"`
    Stock       int     `json:"stock" example:"10" binding:"required"`
    ImageURL    string  `json:"image_url" example:"https://example.com/laptop.jpg"`
}

// Review represents a product review
type Review struct {
    ID        int       `json:"id" example:"1"`
    ProductID int       `json:"product_id" example:"1"`
    UserID    int       `json:"user_id" example:"1"`
    Rating    int       `json:"rating" example:"5"`
    Comment   string    `json:"comment" example:"Great product!"`
    CreatedAt time.Time `json:"created_at" example:"2024-01-15T10:30:00Z"`
}

// CreateReviewRequest is the request body for creating a review
type CreateReviewRequest struct {
    Rating  int    `json:"rating" example:"5" binding:"required"`
    Comment string `json:"comment" example:"Great product!"`
}

// ErrorResponse represents an error response
type ErrorResponse struct {
    Error string `json:"error" example:"Product not found"`
}

// Storage
var (
    products   = make(map[int]Product)
    reviews    = make(map[int]Review)
    nextProdID = 1
    nextRevID  = 1
    productsMu sync.RWMutex
    reviewsMu  sync.RWMutex
)

// @Summary      List products
// @Description  Get a list of products with optional filters
// @Tags         products
// @Accept       json
// @Produce      json
// @Param        category   query     string  false  "Filter by category"
// @Param        min_price  query     number  false  "Minimum price"
// @Param        max_price  query     number  false  "Maximum price"
// @Param        in_stock   query     boolean false  "Only show in-stock items"
// @Param        sort       query     string  false  "Sort by" Enums(name, price, rating)
// @Param        page       query     int     false  "Page number" default(1)
// @Param        limit      query     int     false  "Items per page" default(10)
// @Success      200        {array}   Product
// @Failure      500        {object}  ErrorResponse
// @Router       /products [get]
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

// @Summary      Get product by ID
// @Description  Get a single product by its ID
// @Tags         products
// @Accept       json
// @Produce      json
// @Param        id   path      int  true  "Product ID"
// @Success      200  {object}  Product
// @Failure      404  {object}  ErrorResponse
// @Router       /products/{id} [get]
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
    var req CreateProductRequest
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

// @Summary      Update product
// @Description  Update an existing product (admin only)
// @Tags         products
// @Accept       json
// @Produce      json
// @Param        id       path      int                   true  "Product ID"
// @Param        product  body      CreateProductRequest  true  "Updated product data"
// @Success      200      {object}  Product
// @Failure      400      {object}  ErrorResponse
// @Failure      404      {object}  ErrorResponse
// @Security     BearerAuth
// @Router       /products/{id} [put]
func updateProduct(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    var req CreateProductRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

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

// @Summary      Delete product
// @Description  Delete a product by ID (admin only)
// @Tags         products
// @Accept       json
// @Produce      json
// @Param        id   path      int  true  "Product ID"
// @Success      204
// @Failure      404  {object}  ErrorResponse
// @Security     BearerAuth
// @Router       /products/{id} [delete]
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

// @Summary      Add product review
// @Description  Add a review to a product
// @Tags         reviews
// @Accept       json
// @Produce      json
// @Param        id      path      int                  true  "Product ID"
// @Param        review  body      CreateReviewRequest  true  "Review data"
// @Success      201     {object}  Review
// @Failure      400     {object}  ErrorResponse
// @Failure      404     {object}  ErrorResponse
// @Security     BearerAuth
// @Router       /products/{id}/review [post]
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
    defer r.Body.Close()

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

    // Update product rating
    updateProductRating(productID)

    respondJSON(w, http.StatusCreated, review)
}

// @Summary      List categories
// @Description  Get a list of all product categories
// @Tags         categories
// @Accept       json
// @Produce      json
// @Success      200  {array}   string
// @Failure      500  {object}  ErrorResponse
// @Router       /categories [get]
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

    now := time.Now()

    products[1] = Product{
        ID:          1,
        Name:        "Laptop",
        Description: "High-performance laptop for professionals",
        Price:       999.99,
        Category:    "Electronics",
        Stock:       10,
        ImageURL:    "https://example.com/laptop.jpg",
        Rating:      4.5,
        CreatedAt:   now,
    }

    products[2] = Product{
        ID:          2,
        Name:        "Coffee Mug",
        Description: "Ceramic coffee mug with handle",
        Price:       12.99,
        Category:    "Home",
        Stock:       50,
        ImageURL:    "https://example.com/mug.jpg",
        Rating:      4.0,
        CreatedAt:   now,
    }

    products[3] = Product{
        ID:          3,
        Name:        "Desk Chair",
        Description: "Ergonomic office chair",
        Price:       299.99,
        Category:    "Furniture",
        Stock:       5,
        ImageURL:    "https://example.com/chair.jpg",
        Rating:      4.8,
        CreatedAt:   now,
    }

    nextProdID = 4
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

    println("Server starting on :8080")
    println("API: http://localhost:8080/api/v1")
    println("Swagger UI: http://localhost:8080/swagger/index.html")
    http.ListenAndServe(":8080", r)
}
```

---

## Key Concepts Explained

### 1. General API Information

```go
// @title           Product Catalog API
// @version         1.0
// @description     A product catalog API with categories, reviews, and filtering capabilities
// @termsOfService  http://swagger.io/terms/

// @contact.name   API Support
// @contact.url    http://www.example.com/support
// @contact.email  support@example.com

// @license.name  MIT
// @license.url   https://opensource.org/licenses/MIT

// @host      localhost:8080
// @BasePath  /api/v1
```

**Appears in Swagger UI as**:
- Page title
- Version badge
- Description at top
- Contact info
- License info

### 2. Security Definition

```go
// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization
// @description Type "Bearer" followed by a space and JWT token.
```

**Creates**: "Authorize" button in Swagger UI where users can enter their token.

### 3. Model Documentation with Examples

```go
type Product struct {
    ID          int       `json:"id" example:"1"`
    Name        string    `json:"name" example:"Laptop"`
    Description string    `json:"description" example:"High-performance laptop"`
    Price       float64   `json:"price" example:"999.99"`
    Category    string    `json:"category" example:"Electronics"`
    Stock       int       `json:"stock" example:"10"`
    ImageURL    string    `json:"image_url" example:"https://example.com/laptop.jpg"`
    Rating      float64   `json:"rating" example:"4.5"`
    CreatedAt   time.Time `json:"created_at" example:"2024-01-15T10:30:00Z"`
}
```

**Benefits**:
- Swagger UI shows example values
- "Try it out" pre-fills with examples
- Makes API easier to understand

### 4. Query Parameters with Enums

```go
// @Param        category   query     string  false  "Filter by category"
// @Param        min_price  query     number  false  "Minimum price"
// @Param        max_price  query     number  false  "Maximum price"
// @Param        in_stock   query     boolean false  "Only show in-stock items"
// @Param        sort       query     string  false  "Sort by" Enums(name, price, rating)
// @Param        page       query     int     false  "Page number" default(1)
// @Param        limit      query     int     false  "Items per page" default(10)
```

**Creates**: Form fields in Swagger UI where:
- Enums become dropdowns
- Defaults are pre-filled
- Types are validated

### 5. Path Parameters

```go
// @Param        id   path      int  true  "Product ID"
```

**Format**: `name location type required "description"`
- **location**: `path`, `query`, `header`, `body`
- **type**: `int`, `string`, `number`, `boolean`, `object`, `array`
- **required**: `true` or `false`

### 6. Body Parameters

```go
// @Param        product  body      CreateProductRequest  true  "Product data"
```

**Creates**: JSON editor in Swagger UI with schema based on the struct.

### 7. Success and Failure Responses

```go
// @Success      201      {object}  Product
// @Failure      400      {object}  ErrorResponse
// @Failure      404      {object}  ErrorResponse
```

**Shows**: Expected response structure for each status code.

### 8. Security on Endpoints

```go
// @Security     BearerAuth
```

**Effect**: Padlock icon appears, indicating auth is required.

---

## Generated Documentation Structure

After running `swag init`, you'll have:

```
docs/
  ├── docs.go       # Generated Go code
  ├── swagger.json  # OpenAPI spec in JSON
  └── swagger.yaml  # OpenAPI spec in YAML
```

### swagger.json excerpt:

```json
{
  "swagger": "2.0",
  "info": {
    "title": "Product Catalog API",
    "version": "1.0",
    "description": "A product catalog API..."
  },
  "paths": {
    "/products": {
      "get": {
        "summary": "List products",
        "parameters": [
          {
            "name": "category",
            "in": "query",
            "type": "string"
          }
        ],
        "responses": {
          "200": {
            "description": "OK",
            "schema": {
              "type": "array",
              "items": {
                "$ref": "#/definitions/Product"
              }
            }
          }
        }
      }
    }
  }
}
```

---

## Testing in Swagger UI

### 1. Access Swagger UI
Visit: `http://localhost:8080/swagger/index.html`

### 2. Test Public Endpoints

**GET /products**:
1. Expand the endpoint
2. Click "Try it out"
3. Add filters (optional): `category=Electronics`, `min_price=50`
4. Click "Execute"
5. See response below

### 3. Test Protected Endpoints

**POST /products**:
1. Click "Authorize" button (top right)
2. Enter: `Bearer your-jwt-token-here`
3. Click "Authorize", then "Close"
4. Expand POST /products
5. Click "Try it out"
6. Edit JSON body
7. Click "Execute"

### 4. Test Path Parameters

**GET /products/{id}**:
1. Expand endpoint
2. Click "Try it out"
3. Enter ID: `1`
4. Click "Execute"

---

## Comparison: Before and After

### Before (No Documentation)
```
"Where do I get the list of products?"
"What filters are available?"
"Is authentication required?"
"What's the response format?"
"How do I test this?"
```

### After (With Swagger)
- **Self-service**: All info in one place
- **Interactive**: Test directly in browser
- **Clear**: See all parameters and responses
- **Examples**: Pre-filled values
- **Authentication**: Built-in auth testing

---

## Common Annotations Reference

| Annotation | Purpose | Example |
|------------|---------|---------|
| `@Summary` | Short description | `@Summary Get all users` |
| `@Description` | Long description | `@Description Retrieve...` |
| `@Tags` | Group endpoints | `@Tags users` |
| `@Accept` | Request content type | `@Accept json` |
| `@Produce` | Response content type | `@Produce json` |
| `@Param` | Parameter | `@Param id path int true "ID"` |
| `@Success` | Success response | `@Success 200 {object} User` |
| `@Failure` | Error response | `@Failure 404 {object} Error` |
| `@Router` | Route path | `@Router /users/{id} [get]` |
| `@Security` | Auth required | `@Security BearerAuth` |

---

## What You've Learned

✅ **Swagger annotations** for API info, endpoints, and models  
✅ **Example tags** for sample values  
✅ **Query parameters** with types and descriptions  
✅ **Path parameters** in URL  
✅ **Body parameters** with struct references  
✅ **Security definitions** for authentication  
✅ **Response documentation** for success and errors  
✅ **Interactive testing** in Swagger UI  
✅ **OpenAPI spec generation** automatically from code  

You now have professional, interactive API documentation that developers love!
