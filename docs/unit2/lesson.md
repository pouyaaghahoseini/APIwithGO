# Unit 2: Building HTTP Servers

**Duration**: 60-75 minutes  
**Prerequisites**: Unit 1 (Go fundamentals)  
**Goal**: Build RESTful HTTP servers and understand request/response handling

---

## 2.1 Introduction to HTTP in Go

Go's `net/http` package is one of the best standard library HTTP implementations in any language. You can build production-ready HTTP servers without external frameworks.

### Why Go's HTTP Library is Special

- **Part of standard library** - No external dependencies
- **Production-ready** - Used by Google, Uber, and many others
- **Performant** - Handles thousands of concurrent connections
- **Simple API** - Easy to learn, hard to misuse
- **Middleware-friendly** - Clean composition patterns

---

## 2.2 Your First HTTP Server

### Hello World Server

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    // Register a handler for the root path
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, World!")
    })
    
    // Start the server on port 8080
    fmt.Println("Server starting on http://localhost:8080")
    http.ListenAndServe(":8080", nil)
}
```

**Test it**:
```bash
go run main.go

# In another terminal:
curl http://localhost:8080
# Output: Hello, World!
```

### Understanding the Components

**`http.HandleFunc`** - Registers a handler function for a path
- First parameter: URL path pattern
- Second parameter: Handler function

**Handler Function** - Has this signature:
```go
func(w http.ResponseWriter, r *http.Request)
```
- `w` (ResponseWriter): Write your response here
- `r` (Request): Contains all request information

**`http.ListenAndServe`** - Starts the HTTP server
- First parameter: Address (":8080" means all interfaces, port 8080)
- Second parameter: Handler (nil uses DefaultServeMux)

---

## 2.3 The Request Object

The `http.Request` struct contains everything about the incoming request.

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // HTTP Method
    method := r.Method  // "GET", "POST", "PUT", etc.
    
    // URL and Path
    path := r.URL.Path        // "/users/123"
    rawQuery := r.URL.RawQuery  // "page=1&limit=10"
    
    // Headers
    userAgent := r.Header.Get("User-Agent")
    contentType := r.Header.Get("Content-Type")
    
    // Query Parameters
    page := r.URL.Query().Get("page")
    limit := r.URL.Query().Get("limit")
    
    // Request Body (for POST/PUT)
    // body, err := io.ReadAll(r.Body)
    // defer r.Body.Close()
    
    // Remote Address
    clientIP := r.RemoteAddr
    
    // Cookies
    cookie, err := r.Cookie("session_id")
}
```

### Complete Example

```go
package main

import (
    "fmt"
    "net/http"
)

func infoHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Method: %s\n", r.Method)
    fmt.Fprintf(w, "Path: %s\n", r.URL.Path)
    fmt.Fprintf(w, "Query: %s\n", r.URL.RawQuery)
    fmt.Fprintf(w, "User-Agent: %s\n", r.Header.Get("User-Agent"))
    
    // Parse and display query parameters
    fmt.Fprintf(w, "\nQuery Parameters:\n")
    for key, values := range r.URL.Query() {
        for _, value := range values {
            fmt.Fprintf(w, "  %s = %s\n", key, value)
        }
    }
}

func main() {
    http.HandleFunc("/info", infoHandler)
    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", nil)
}
```

**Test it**:
```bash
curl "http://localhost:8080/info?name=John&age=30"
```

---

## 2.4 The ResponseWriter

The `http.ResponseWriter` interface is used to construct the HTTP response.

### Basic Response Writing

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Write plain text
    w.Write([]byte("Hello"))
    
    // Or use fmt.Fprintf
    fmt.Fprintf(w, "Hello, %s", name)
    
    // Or WriteString
    w.WriteString("Hello")
}
```

### Setting Response Headers

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Set headers BEFORE writing body
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("X-Custom-Header", "value")
    
    // Write response
    w.Write([]byte(`{"message": "Hello"}`))
}
```

### Setting Status Codes

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Set status code (must be before Write)
    w.WriteHeader(http.StatusCreated)  // 201
    w.Write([]byte("Resource created"))
}

// Common status codes
http.StatusOK                  // 200
http.StatusCreated             // 201
http.StatusNoContent           // 204
http.StatusBadRequest          // 400
http.StatusUnauthorized        // 401
http.StatusForbidden           // 403
http.StatusNotFound            // 404
http.StatusInternalServerError // 500
```

**Important**: Call `WriteHeader` before any `Write` calls. Once you write the body, you can't change headers or status code.

---

## 2.5 Handling JSON

Most APIs use JSON for request and response bodies.

### Sending JSON Responses

```go
package main

import (
    "encoding/json"
    "net/http"
)

type User struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
}

func getUserHandler(w http.ResponseWriter, r *http.Request) {
    user := User{
        ID:       1,
        Username: "john_doe",
        Email:    "john@example.com",
    }
    
    // Set content type
    w.Header().Set("Content-Type", "application/json")
    
    // Encode and send
    json.NewEncoder(w).Encode(user)
}

func main() {
    http.HandleFunc("/user", getUserHandler)
    http.ListenAndServe(":8080", nil)
}
```

### Receiving JSON Requests

```go
type CreateUserRequest struct {
    Username string `json:"username"`
    Email    string `json:"email"`
    Password string `json:"password"`
}

func createUserHandler(w http.ResponseWriter, r *http.Request) {
    // Only accept POST
    if r.Method != http.MethodPost {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }
    
    // Decode JSON body
    var req CreateUserRequest
    err := json.NewDecoder(r.Body).Decode(&req)
    if err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    defer r.Body.Close()
    
    // Validate
    if req.Username == "" || req.Email == "" {
        http.Error(w, "Missing required fields", http.StatusBadRequest)
        return
    }
    
    // Process request (create user in database, etc.)
    
    // Send response
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(map[string]interface{}{
        "id":       1,
        "username": req.Username,
        "email":    req.Email,
    })
}
```

### Helper Function for JSON Responses

```go
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}

// Usage
func handler(w http.ResponseWriter, r *http.Request) {
    user := getUser()
    if user == nil {
        respondError(w, http.StatusNotFound, "User not found")
        return
    }
    respondJSON(w, http.StatusOK, user)
}
```

---

## 2.6 Routing with Standard Library

The standard library supports basic routing:

```go
func main() {
    http.HandleFunc("/", homeHandler)
    http.HandleFunc("/users", usersHandler)
    http.HandleFunc("/products", productsHandler)
    
    http.ListenAndServe(":8080", nil)
}
```

### Handling Different HTTP Methods

```go
func usersHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        // GET /users - list users
        getUsers(w, r)
    case http.MethodPost:
        // POST /users - create user
        createUser(w, r)
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}
```

### Limitations of Standard Library Routing

- No URL parameters (e.g., `/users/{id}`)
- No pattern matching
- No route grouping
- No middleware chaining

**Solution**: Use a router library like Gorilla Mux or Chi.

---

## 2.7 Advanced Routing with Gorilla Mux

Gorilla Mux is the most popular Go router, providing advanced routing features.

### Installation

```bash
go get github.com/gorilla/mux
```

### Basic Usage

```go
package main

import (
    "encoding/json"
    "net/http"
    
    "github.com/gorilla/mux"
)

func main() {
    r := mux.NewRouter()
    
    // Basic routes
    r.HandleFunc("/", homeHandler).Methods("GET")
    r.HandleFunc("/users", getUsers).Methods("GET")
    r.HandleFunc("/users", createUser).Methods("POST")
    
    // Route with URL parameter
    r.HandleFunc("/users/{id}", getUser).Methods("GET")
    r.HandleFunc("/users/{id}", updateUser).Methods("PUT")
    r.HandleFunc("/users/{id}", deleteUser).Methods("DELETE")
    
    http.ListenAndServe(":8080", r)
}
```

### Extracting URL Parameters

```go
func getUser(w http.ResponseWriter, r *http.Request) {
    // Extract {id} from URL
    vars := mux.Vars(r)
    userID := vars["id"]
    
    // Convert to int if needed
    id, err := strconv.Atoi(userID)
    if err != nil {
        http.Error(w, "Invalid user ID", http.StatusBadRequest)
        return
    }
    
    // Fetch user with id...
}
```

### Route Parameters and Query Strings

```go
// URL: /users/123?include=profile&format=json
func getUser(w http.ResponseWriter, r *http.Request) {
    // Path parameter
    vars := mux.Vars(r)
    userID := vars["id"]  // "123"
    
    // Query parameters
    include := r.URL.Query().Get("include")  // "profile"
    format := r.URL.Query().Get("format")    // "json"
}
```

### Route Groups (Subrouters)

```go
func main() {
    r := mux.NewRouter()
    
    // API v1 routes
    api := r.PathPrefix("/api/v1").Subrouter()
    api.HandleFunc("/users", getUsers).Methods("GET")
    api.HandleFunc("/users/{id}", getUser).Methods("GET")
    api.HandleFunc("/products", getProducts).Methods("GET")
    
    // Admin routes
    admin := r.PathPrefix("/admin").Subrouter()
    admin.HandleFunc("/users", adminGetUsers).Methods("GET")
    admin.HandleFunc("/stats", getStats).Methods("GET")
    
    http.ListenAndServe(":8080", r)
}
```

### Route Matching with Regular Expressions

```go
// Only accept numeric IDs
r.HandleFunc("/users/{id:[0-9]+}", getUser).Methods("GET")

// Match specific patterns
r.HandleFunc("/files/{filename:.+\\.pdf}", getFile).Methods("GET")
```

---

## 2.8 Building a RESTful API

RESTful APIs follow conventions for resource operations.

### REST Conventions

| Method | Path | Action | Response |
|--------|------|--------|----------|
| GET | /users | List all users | 200 + array |
| GET | /users/{id} | Get one user | 200 + object |
| POST | /users | Create user | 201 + object |
| PUT | /users/{id} | Update user | 200 + object |
| DELETE | /users/{id} | Delete user | 204 (no content) |

### Complete REST API Example

```go
package main

import (
    "encoding/json"
    "net/http"
    "strconv"
    "sync"
    
    "github.com/gorilla/mux"
)

type User struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
}

// In-memory storage (use database in production)
var (
    users   = make(map[int]User)
    nextID  = 1
    usersMu sync.RWMutex
)

// GET /users - List all users
func getUsers(w http.ResponseWriter, r *http.Request) {
    usersMu.RLock()
    defer usersMu.RUnlock()
    
    userList := make([]User, 0, len(users))
    for _, user := range users {
        userList = append(userList, user)
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(userList)
}

// GET /users/{id} - Get single user
func getUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        http.Error(w, "Invalid user ID", http.StatusBadRequest)
        return
    }
    
    usersMu.RLock()
    user, exists := users[id]
    usersMu.RUnlock()
    
    if !exists {
        http.Error(w, "User not found", http.StatusNotFound)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

// POST /users - Create user
func createUser(w http.ResponseWriter, r *http.Request) {
    var user User
    err := json.NewDecoder(r.Body).Decode(&user)
    if err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    defer r.Body.Close()
    
    // Validation
    if user.Username == "" || user.Email == "" {
        http.Error(w, "Username and email are required", http.StatusBadRequest)
        return
    }
    
    // Assign ID and store
    usersMu.Lock()
    user.ID = nextID
    nextID++
    users[user.ID] = user
    usersMu.Unlock()
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}

// PUT /users/{id} - Update user
func updateUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        http.Error(w, "Invalid user ID", http.StatusBadRequest)
        return
    }
    
    var user User
    err = json.NewDecoder(r.Body).Decode(&user)
    if err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    defer r.Body.Close()
    
    usersMu.Lock()
    _, exists := users[id]
    if !exists {
        usersMu.Unlock()
        http.Error(w, "User not found", http.StatusNotFound)
        return
    }
    
    user.ID = id
    users[id] = user
    usersMu.Unlock()
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

// DELETE /users/{id} - Delete user
func deleteUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        http.Error(w, "Invalid user ID", http.StatusBadRequest)
        return
    }
    
    usersMu.Lock()
    _, exists := users[id]
    if !exists {
        usersMu.Unlock()
        http.Error(w, "User not found", http.StatusNotFound)
        return
    }
    
    delete(users, id)
    usersMu.Unlock()
    
    w.WriteHeader(http.StatusNoContent)
}

func main() {
    r := mux.NewRouter()
    
    // User routes
    r.HandleFunc("/users", getUsers).Methods("GET")
    r.HandleFunc("/users/{id}", getUser).Methods("GET")
    r.HandleFunc("/users", createUser).Methods("POST")
    r.HandleFunc("/users/{id}", updateUser).Methods("PUT")
    r.HandleFunc("/users/{id}", deleteUser).Methods("DELETE")
    
    http.ListenAndServe(":8080", r)
}
```

**Test it**:
```bash
# Create users
curl -X POST http://localhost:8080/users \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","email":"alice@example.com"}'

curl -X POST http://localhost:8080/users \
  -H "Content-Type: application/json" \
  -d '{"username":"bob","email":"bob@example.com"}'

# Get all users
curl http://localhost:8080/users

# Get single user
curl http://localhost:8080/users/1

# Update user
curl -X PUT http://localhost:8080/users/1 \
  -H "Content-Type: application/json" \
  -d '{"username":"alice_updated","email":"alice@example.com"}'

# Delete user
curl -X DELETE http://localhost:8080/users/2
```

---

## 2.9 Middleware

Middleware are functions that run before your handlers, useful for logging, authentication, CORS, etc.

### Basic Middleware Pattern

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Before handler
        fmt.Printf("%s %s\n", r.Method, r.URL.Path)
        
        // Call next handler
        next.ServeHTTP(w, r)
        
        // After handler (if needed)
    })
}
```

### Applying Middleware with Mux

```go
func main() {
    r := mux.NewRouter()
    
    r.HandleFunc("/users", getUsers).Methods("GET")
    r.HandleFunc("/users/{id}", getUser).Methods("GET")
    
    // Apply middleware to all routes
    r.Use(loggingMiddleware)
    
    http.ListenAndServe(":8080", r)
}
```

### Multiple Middleware

```go
func main() {
    r := mux.NewRouter()
    
    r.HandleFunc("/users", getUsers).Methods("GET")
    
    // Apply multiple middleware (executed in order)
    r.Use(loggingMiddleware)
    r.Use(corsMiddleware)
    r.Use(authMiddleware)
    
    http.ListenAndServe(":8080", r)
}
```

### Common Middleware Examples

#### Logging Middleware

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Call handler
        next.ServeHTTP(w, r)
        
        // Log after
        fmt.Printf("[%s] %s %s - %v\n",
            time.Now().Format("2006-01-02 15:04:05"),
            r.Method,
            r.URL.Path,
            time.Since(start))
    })
}
```

#### CORS Middleware

```go
func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        
        // Handle preflight
        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}
```

#### Content-Type Middleware

```go
func jsonMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        next.ServeHTTP(w, r)
    })
}
```

#### Recovery Middleware (Panic Recovery)

```go
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                fmt.Printf("Panic recovered: %v\n", err)
                http.Error(w, "Internal server error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

### Middleware for Specific Routes

```go
func main() {
    r := mux.NewRouter()
    
    // Public routes (no auth)
    r.HandleFunc("/login", login).Methods("POST")
    r.HandleFunc("/register", register).Methods("POST")
    
    // Protected routes
    protected := r.PathPrefix("/api").Subrouter()
    protected.Use(authMiddleware)  // Only these routes need auth
    protected.HandleFunc("/users", getUsers).Methods("GET")
    protected.HandleFunc("/profile", getProfile).Methods("GET")
    
    http.ListenAndServe(":8080", r)
}
```

---

## 2.10 Organizing Your API

As your API grows, organize code into packages.

### Recommended Structure

```
myapi/
├── main.go
├── go.mod
├── go.sum
├── handlers/
│   ├── user.go
│   ├── product.go
│   └── auth.go
├── models/
│   ├── user.go
│   └── product.go
├── middleware/
│   ├── logging.go
│   ├── auth.go
│   └── cors.go
├── utils/
│   └── response.go
└── config/
    └── config.go
```

### Example: handlers/user.go

```go
package handlers

import (
    "encoding/json"
    "net/http"
    
    "myapi/models"
    "github.com/gorilla/mux"
)

type UserHandler struct {
    // Dependencies (database, services, etc.)
}

func NewUserHandler() *UserHandler {
    return &UserHandler{}
}

func (h *UserHandler) GetUsers(w http.ResponseWriter, r *http.Request) {
    users := []models.User{
        {ID: 1, Username: "alice", Email: "alice@example.com"},
        {ID: 2, Username: "bob", Email: "bob@example.com"},
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    userID := vars["id"]
    
    // Fetch user...
    user := models.User{ID: 1, Username: "alice", Email: "alice@example.com"}
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var user models.User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    defer r.Body.Close()
    
    // Create user...
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}
```

### Example: models/user.go

```go
package models

type User struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
}

type CreateUserRequest struct {
    Username string `json:"username" validate:"required"`
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=8"`
}
```

### Example: utils/response.go

```go
package utils

import (
    "encoding/json"
    "net/http"
)

func RespondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func RespondError(w http.ResponseWriter, status int, message string) {
    RespondJSON(w, status, map[string]string{"error": message})
}
```

### Example: main.go

```go
package main

import (
    "fmt"
    "net/http"
    
    "myapi/handlers"
    "myapi/middleware"
    
    "github.com/gorilla/mux"
)

func main() {
    r := mux.NewRouter()
    
    // Initialize handlers
    userHandler := handlers.NewUserHandler()
    
    // Apply global middleware
    r.Use(middleware.LoggingMiddleware)
    r.Use(middleware.CorsMiddleware)
    
    // Public routes
    r.HandleFunc("/health", healthCheck).Methods("GET")
    
    // API routes
    api := r.PathPrefix("/api/v1").Subrouter()
    api.HandleFunc("/users", userHandler.GetUsers).Methods("GET")
    api.HandleFunc("/users/{id}", userHandler.GetUser).Methods("GET")
    api.HandleFunc("/users", userHandler.CreateUser).Methods("POST")
    
    // Start server
    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", r)
}

func healthCheck(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("OK"))
}
```

---

## 2.11 Error Handling Patterns

### Standard Error Response

```go
type ErrorResponse struct {
    Error   string `json:"error"`
    Message string `json:"message,omitempty"`
    Code    string `json:"code,omitempty"`
}

func respondError(w http.ResponseWriter, status int, err string, message string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(ErrorResponse{
        Error:   err,
        Message: message,
    })
}

// Usage
func handler(w http.ResponseWriter, r *http.Request) {
    user, err := getUser(id)
    if err != nil {
        respondError(w, http.StatusNotFound, "not_found", "User not found")
        return
    }
    // ...
}
```

### Validation Error Response

```go
type ValidationError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

type ValidationErrorResponse struct {
    Error  string            `json:"error"`
    Errors []ValidationError `json:"errors"`
}

func respondValidationErrors(w http.ResponseWriter, errors []ValidationError) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusBadRequest)
    json.NewEncoder(w).Encode(ValidationErrorResponse{
        Error:  "validation_failed",
        Errors: errors,
    })
}
```

---

## 2.12 Testing HTTP Handlers

Go makes it easy to test HTTP handlers.

```go
package handlers

import (
    "net/http"
    "net/http/httptest"
    "testing"
)

func TestGetUsers(t *testing.T) {
    // Create request
    req, err := http.NewRequest("GET", "/users", nil)
    if err != nil {
        t.Fatal(err)
    }
    
    // Create response recorder
    rr := httptest.NewRecorder()
    
    // Create handler
    handler := http.HandlerFunc(GetUsers)
    
    // Serve request
    handler.ServeHTTP(rr, req)
    
    // Check status code
    if status := rr.Code; status != http.StatusOK {
        t.Errorf("handler returned wrong status code: got %v want %v",
            status, http.StatusOK)
    }
    
    // Check response body
    expected := `[{"id":1,"username":"alice"}]`
    if rr.Body.String() != expected {
        t.Errorf("handler returned unexpected body: got %v want %v",
            rr.Body.String(), expected)
    }
}
```

---

## Key Takeaways

✅ **http.Handler interface** - Core of Go's HTTP handling  
✅ **Request and ResponseWriter** - How to read requests and write responses  
✅ **JSON handling** - Encoding/decoding for API communication  
✅ **Gorilla Mux** - Advanced routing with URL parameters  
✅ **REST conventions** - Standard patterns for resource operations  
✅ **Middleware** - Composable request processing  
✅ **Code organization** - Structuring larger APIs  
✅ **Error handling** - Consistent error responses  

---

## Best Practices

1. **Always set Content-Type** header before writing response
2. **Call WriteHeader before Write** - can't change status after writing
3. **Close request bodies** with defer when reading
4. **Use struct handlers** for dependency injection
5. **Validate input** before processing
6. **Return proper HTTP status codes**
7. **Use middleware** for cross-cutting concerns
8. **Organize code** into packages as it grows

---

## Common Status Codes Reference

**Success (2xx)**:
- 200 OK - Request succeeded
- 201 Created - Resource created
- 204 No Content - Success with no response body

**Client Error (4xx)**:
- 400 Bad Request - Invalid request
- 401 Unauthorized - Authentication required
- 403 Forbidden - Not allowed
- 404 Not Found - Resource doesn't exist
- 422 Unprocessable Entity - Validation failed

**Server Error (5xx)**:
- 500 Internal Server Error - Server error
- 503 Service Unavailable - Server overloaded

---

## What's Next?

Now that you can build HTTP servers and handle requests/responses, Unit 3 will cover:
- Authentication with JWT
- Password hashing
- Protected routes
- Authorization middleware

You're ready to build secure APIs! 🚀
