# Unit 5: API Documentation with Swagger/OpenAPI

**Duration**: 60-75 minutes  
**Prerequisites**: Units 1-4 (Go fundamentals, HTTP servers, Authentication, Versioning)  
**Goal**: Generate interactive API documentation automatically from code

---

## 5.1 Why API Documentation Matters

**Without documentation**:
- Developers waste time guessing endpoint behavior
- Support tickets increase
- Integration takes longer
- API adoption suffers

**With good documentation**:
- Self-service integration
- Reduced support burden
- Faster onboarding
- Higher API adoption
- Better developer experience

---

## 5.2 What is Swagger/OpenAPI?

**OpenAPI Specification (OAS)**: A standard format for describing REST APIs in JSON or YAML.

**Swagger**: A set of tools built around OpenAPI:
- **Swagger UI**: Interactive documentation interface
- **Swagger Codegen**: Generate client SDKs
- **Swagger Editor**: Design APIs visually

**Current Version**: OpenAPI 3.0 (we'll use this)

---

## 5.3 OpenAPI Specification Structure

### Basic Structure

```yaml
openapi: 3.0.0
info:
  title: My API
  version: 1.0.0
  description: API description
servers:
  - url: http://localhost:8080
    description: Development server
paths:
  /users:
    get:
      summary: List all users
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/User'
components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
        name:
          type: string
        email:
          type: string
```

### Key Sections

1. **info**: API metadata (title, version, description)
2. **servers**: Base URLs for different environments
3. **paths**: API endpoints and operations
4. **components**: Reusable schemas, parameters, responses
5. **security**: Authentication/authorization schemes

---

## 5.4 Documenting APIs in Go

### Popular Libraries

1. **swaggo/swag** (Most popular)
   - Annotations in code
   - Automatic generation
   - Swagger UI integration

2. **go-swagger**
   - More powerful
   - Steeper learning curve

We'll use **swaggo/swag** for this course.

---

## 5.5 Installing Swag

```bash
# Install swag CLI
go install github.com/swaggo/swag/cmd/swag@latest

# Install required packages
go get -u github.com/swaggo/http-swagger
go get -u github.com/swaggo/files
```

---

## 5.6 Basic API Documentation

### Step 1: Add General API Info

Add comments to your `main.go`:

```go
package main

import (
    "net/http"
    
    httpSwagger "github.com/swaggo/http-swagger"
    _ "myapi/docs"  // Import generated docs
)

// @title           User Management API
// @version         1.0
// @description     A simple user management API with CRUD operations
// @termsOfService  http://swagger.io/terms/

// @contact.name   API Support
// @contact.url    http://www.swagger.io/support
// @contact.email  support@swagger.io

// @license.name  Apache 2.0
// @license.url   http://www.apache.org/licenses/LICENSE-2.0.html

// @host      localhost:8080
// @BasePath  /api/v1

// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization
// @description Type "Bearer" followed by a space and JWT token.

func main() {
    r := mux.NewRouter()
    
    // Swagger UI endpoint
    r.PathPrefix("/swagger/").Handler(httpSwagger.WrapHandler)
    
    // Your API routes
    r.HandleFunc("/api/v1/users", getUsers).Methods("GET")
    
    http.ListenAndServe(":8080", r)
}
```

### Step 2: Document Handlers

```go
// @Summary      Get all users
// @Description  Retrieve a list of all users
// @Tags         users
// @Accept       json
// @Produce      json
// @Success      200  {array}   User
// @Failure      500  {object}  ErrorResponse
// @Router       /users [get]
func getUsers(w http.ResponseWriter, r *http.Request) {
    // Implementation
}

// @Summary      Get user by ID
// @Description  Get a single user by their ID
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        id   path      int  true  "User ID"
// @Success      200  {object}  User
// @Failure      404  {object}  ErrorResponse
// @Router       /users/{id} [get]
func getUser(w http.ResponseWriter, r *http.Request) {
    // Implementation
}

// @Summary      Create user
// @Description  Create a new user
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        user  body      CreateUserRequest  true  "User data"
// @Success      201   {object}  User
// @Failure      400   {object}  ErrorResponse
// @Security     BearerAuth
// @Router       /users [post]
func createUser(w http.ResponseWriter, r *http.Request) {
    // Implementation
}

// @Summary      Delete user
// @Description  Delete a user by ID
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        id   path      int  true  "User ID"
// @Success      204
// @Failure      404  {object}  ErrorResponse
// @Security     BearerAuth
// @Router       /users/{id} [delete]
func deleteUser(w http.ResponseWriter, r *http.Request) {
    // Implementation
}
```

### Step 3: Document Models

```go
// User represents a user in the system
type User struct {
    ID        int       `json:"id" example:"1"`
    Username  string    `json:"username" example:"johndoe"`
    Email     string    `json:"email" example:"john@example.com"`
    FullName  string    `json:"full_name" example:"John Doe"`
    CreatedAt time.Time `json:"created_at" example:"2024-01-15T10:30:00Z"`
}

// CreateUserRequest is the request body for creating a user
type CreateUserRequest struct {
    Username string `json:"username" example:"johndoe" binding:"required"`
    Email    string `json:"email" example:"john@example.com" binding:"required"`
    Password string `json:"password" example:"SecurePass123" binding:"required"`
    FullName string `json:"full_name" example:"John Doe"`
}

// ErrorResponse represents an error response
type ErrorResponse struct {
    Error string `json:"error" example:"Invalid request"`
}
```

### Step 4: Generate Documentation

```bash
# Generate docs
swag init

# This creates:
# - docs/docs.go
# - docs/swagger.json
# - docs/swagger.yaml
```

### Step 5: Access Swagger UI

Start your server and visit:
```
http://localhost:8080/swagger/index.html
```

You'll see an interactive API documentation page!

---

## 5.7 Common Annotations

### Handler Annotations

```go
// @Summary      Short description (1 line)
// @Description  Longer description (multiple lines)
// @Tags         grouping-tag
// @Accept       json|xml|plain|html|mpfd|x-www-form-urlencoded
// @Produce      json|xml|plain|html
// @Param        name  location  type  required  "description"
// @Success      code  {type}    Model
// @Failure      code  {type}    Model
// @Router       /path [method]
// @Security     SecurityScheme
```

### Parameter Locations

- **path**: URL parameter (`/users/{id}`)
- **query**: Query string (`/users?name=john`)
- **header**: HTTP header
- **body**: Request body
- **formData**: Form data

### Examples

```go
// Path parameter
// @Param id path int true "User ID"

// Query parameter
// @Param name query string false "User name"

// Header parameter
// @Param Authorization header string true "Bearer token"

// Body parameter
// @Param user body CreateUserRequest true "User data"

// Multiple query parameters
// @Param page query int false "Page number" default(1)
// @Param limit query int false "Items per page" default(10)
```

---

## 5.8 Advanced Features

### Array Responses

```go
// @Success 200 {array} User
```

### Nested Objects

```go
type Post struct {
    ID     int    `json:"id"`
    Title  string `json:"title"`
    Author User   `json:"author"`  // Nested
}

// @Success 200 {object} Post
```

### Enumerations

```go
type Status string

const (
    StatusActive   Status = "active"
    StatusInactive Status = "inactive"
)

type User struct {
    Status Status `json:"status" enums:"active,inactive"`
}
```

### Optional vs Required

```go
type UpdateUserRequest struct {
    FullName string `json:"full_name" example:"John Doe"`           // Optional
    Email    string `json:"email" example:"john@example.com" binding:"required"`  // Required
}
```

---

## 5.9 Authentication in Swagger

### Bearer Token (JWT)

```go
// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization
// @description Type "Bearer" followed by a space and JWT token.
```

Then on protected endpoints:

```go
// @Security BearerAuth
```

### API Key

```go
// @securityDefinitions.apikey ApiKeyAuth
// @in header
// @name X-API-Key
```

### Basic Auth

```go
// @securityDefinitions.basic BasicAuth
```

### Using in Swagger UI

1. Click "Authorize" button
2. Enter your token/credentials
3. All subsequent requests include auth

---

## 5.10 Multiple Versions

Document multiple API versions:

```go
// main.go for v1
// @BasePath /api/v1

// main.go for v2
// @BasePath /api/v2
```

Or create separate documentation:

```bash
swag init --generalInfo v1/main.go --output docs/v1
swag init --generalInfo v2/main.go --output docs/v2
```

---

## 5.11 Complete Example

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strconv"
    "sync"
    "time"

    "github.com/gorilla/mux"
    httpSwagger "github.com/swaggo/http-swagger"

    _ "myapi/docs"
)

// @title           User Management API
// @version         1.0
// @description     A user management API with CRUD operations
// @termsOfService  http://swagger.io/terms/

// @contact.name   API Support
// @contact.email  support@example.com

// @license.name  MIT
// @license.url   https://opensource.org/licenses/MIT

// @host      localhost:8080
// @BasePath  /api/v1

// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization

// User represents a user in the system
type User struct {
    ID        int       `json:"id" example:"1"`
    Username  string    `json:"username" example:"johndoe"`
    Email     string    `json:"email" example:"john@example.com"`
    FullName  string    `json:"full_name" example:"John Doe"`
    CreatedAt time.Time `json:"created_at" example:"2024-01-15T10:30:00Z"`
}

// CreateUserRequest is the request body for creating a user
type CreateUserRequest struct {
    Username string `json:"username" example:"johndoe" binding:"required"`
    Email    string `json:"email" example:"john@example.com" binding:"required"`
    Password string `json:"password" example:"SecurePass123" binding:"required"`
    FullName string `json:"full_name" example:"John Doe"`
}

// ErrorResponse represents an error
type ErrorResponse struct {
    Error string `json:"error" example:"Invalid request"`
}

// Storage
var (
    users      = make(map[int]User)
    nextUserID = 1
    usersMu    sync.RWMutex
)

// @Summary      List all users
// @Description  Get a list of all users
// @Tags         users
// @Accept       json
// @Produce      json
// @Success      200  {array}   User
// @Failure      500  {object}  ErrorResponse
// @Router       /users [get]
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

// @Summary      Get user by ID
// @Description  Get a single user by their ID
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        id   path      int  true  "User ID"
// @Success      200  {object}  User
// @Failure      404  {object}  ErrorResponse
// @Router       /users/{id} [get]
func getUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    usersMu.RLock()
    user, exists := users[id]
    usersMu.RUnlock()

    if !exists {
        w.WriteHeader(http.StatusNotFound)
        json.NewEncoder(w).Encode(ErrorResponse{Error: "User not found"})
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

// @Summary      Create user
// @Description  Create a new user
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        user  body      CreateUserRequest  true  "User data"
// @Success      201   {object}  User
// @Failure      400   {object}  ErrorResponse
// @Security     BearerAuth
// @Router       /users [post]
func createUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        w.WriteHeader(http.StatusBadRequest)
        json.NewEncoder(w).Encode(ErrorResponse{Error: "Invalid JSON"})
        return
    }

    usersMu.Lock()
    user := User{
        ID:        nextUserID,
        Username:  req.Username,
        Email:     req.Email,
        FullName:  req.FullName,
        CreatedAt: time.Now(),
    }
    users[nextUserID] = user
    nextUserID++
    usersMu.Unlock()

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}

// @Summary      Delete user
// @Description  Delete a user by ID
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        id   path      int  true  "User ID"
// @Success      204
// @Failure      404  {object}  ErrorResponse
// @Security     BearerAuth
// @Router       /users/{id} [delete]
func deleteUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    usersMu.Lock()
    _, exists := users[id]
    if exists {
        delete(users, id)
    }
    usersMu.Unlock()

    if !exists {
        w.WriteHeader(http.StatusNotFound)
        json.NewEncoder(w).Encode(ErrorResponse{Error: "User not found"})
        return
    }

    w.WriteHeader(http.StatusNoContent)
}

func main() {
    r := mux.NewRouter()

    // API routes
    api := r.PathPrefix("/api/v1").Subrouter()
    api.HandleFunc("/users", getUsers).Methods("GET")
    api.HandleFunc("/users/{id}", getUser).Methods("GET")
    api.HandleFunc("/users", createUser).Methods("POST")
    api.HandleFunc("/users/{id}", deleteUser).Methods("DELETE")

    // Swagger UI
    r.PathPrefix("/swagger/").Handler(httpSwagger.WrapHandler)

    fmt.Println("Server starting on :8080")
    fmt.Println("Swagger UI: http://localhost:8080/swagger/index.html")
    http.ListenAndServe(":8080", r)
}
```

**Generate docs**:
```bash
swag init
go run main.go
```

Visit: `http://localhost:8080/swagger/index.html`

---

## 5.12 Best Practices

### ✅ DO

1. **Keep descriptions clear and concise**
2. **Provide example values** for all fields
3. **Document all error responses**
4. **Group related endpoints** with tags
5. **Include authentication requirements**
6. **Update docs when API changes**
7. **Use consistent naming**
8. **Document query parameters** with defaults
9. **Provide sample requests**

### ❌ DON'T

1. **Don't leave descriptions empty**
2. **Don't forget to regenerate** after code changes
3. **Don't document internal endpoints**
4. **Don't use unclear names**
5. **Don't skip authentication docs**
6. **Don't duplicate information**

---

## 5.13 Swagger UI Features

### Interactive Testing

1. **Try it out**: Test endpoints directly
2. **Authorization**: Add tokens/credentials
3. **Request/Response**: See actual data
4. **Code samples**: Generated client code

### Filtering

- Filter by tags
- Search endpoints
- Collapse/expand sections

### Export

- Download OpenAPI spec (JSON/YAML)
- Import into Postman
- Generate client SDKs

---

## 5.14 Alternative Documentation Tools

### Postman

- Import OpenAPI spec
- Create collections
- Share with team

### ReDoc

- More visually appealing
- Better for reading
- Less interactive

```go
import "github.com/go-openapi/runtime/middleware"

r.Handle("/docs", middleware.Redoc(middleware.RedocOpts{
    SpecURL: "/swagger.json",
}, nil))
```

### Stoplight

- Design-first approach
- Collaboration features
- Mock servers

---

## 5.15 Continuous Documentation

### In CI/CD Pipeline

```yaml
# .github/workflows/docs.yml
name: Generate API Docs

on: [push]

jobs:
  docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Set up Go
        uses: actions/setup-go@v2
        
      - name: Install swag
        run: go install github.com/swaggo/swag/cmd/swag@latest
        
      - name: Generate docs
        run: swag init
        
      - name: Deploy docs
        # Deploy to S3, GitHub Pages, etc.
```

### Documentation Versioning

Keep docs in sync with API versions:

```
docs/
  v1/
    swagger.json
    swagger.yaml
  v2/
    swagger.json
    swagger.yaml
```

---

## Key Takeaways

✅ **Swagger/OpenAPI** is the industry standard for API documentation  
✅ **swaggo/swag** generates docs from code annotations  
✅ **Interactive UI** lets developers test APIs  
✅ **Annotations** describe endpoints, parameters, responses  
✅ **Models** document request/response structures  
✅ **Authentication** can be tested in Swagger UI  
✅ **Keep docs updated** - regenerate after changes  
✅ **Best developer experience** comes from good documentation  

---

## What's Next?

Unit 6 will cover Caching Strategies to improve API performance and reduce database load! 🚀
