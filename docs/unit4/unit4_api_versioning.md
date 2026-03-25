# Unit 4: API Versioning

**Duration**: 45-60 minutes  
**Prerequisites**: Unit 1-3 (Go fundamentals, HTTP servers, Authentication)  
**Goal**: Learn strategies for evolving APIs without breaking existing clients

---

## 4.1 Why API Versioning?

APIs evolve over time. You need to:
- Add new features
- Change data structures
- Fix design mistakes
- Improve performance

**The Problem**: Existing clients depend on your current API structure. Breaking changes will crash their applications.

**The Solution**: API versioning allows you to introduce changes while maintaining backward compatibility.

---

## 4.2 What Constitutes a Breaking Change?

### Breaking Changes (Require New Version)

❌ **Removing a field**
```json
// v1
{"id": 1, "name": "John", "email": "john@example.com"}

// v2 - BREAKING!
{"id": 1, "name": "John"}  // email removed
```

❌ **Renaming a field**
```json
// v1
{"user_name": "john"}

// v2 - BREAKING!
{"username": "john"}  // field renamed
```

❌ **Changing field types**
```json
// v1
{"price": 19.99}

// v2 - BREAKING!
{"price": "19.99"}  // number → string
```

❌ **Removing an endpoint**
```
DELETE /api/users/{id}  // No longer available
```

❌ **Changing authentication requirements**
```
// v1: No auth required
GET /api/posts

// v2: Auth required
GET /api/posts  // Now returns 401 without token
```

❌ **Changing response structure**
```json
// v1
{"users": [...]}

// v2 - BREAKING!
{"data": {"users": [...]}}  // Wrapped in data object
```

### Non-Breaking Changes (Safe to Deploy)

✅ **Adding optional fields**
```json
// v1
{"id": 1, "name": "John"}

// v1.1 - Safe!
{"id": 1, "name": "John", "avatar": "url"}  // New optional field
```

✅ **Adding new endpoints**
```
POST /api/users/{id}/avatar  // New endpoint
```

✅ **Adding optional query parameters**
```
GET /api/posts?sort=date  // New optional parameter
```

✅ **Making required fields optional**
```go
// Before: email required
// After: email optional (safer, more lenient)
```

✅ **Expanding enum values**
```go
// Before: status: "active" | "inactive"
// After: status: "active" | "inactive" | "pending"  // Added new value
```

---

## 4.3 Common Versioning Strategies

### Strategy 1: URL Path Versioning (Most Common)

**Pattern**: `/api/v1/`, `/api/v2/`

```go
func main() {
    r := mux.NewRouter()
    
    // Version 1
    v1 := r.PathPrefix("/api/v1").Subrouter()
    v1.HandleFunc("/users", getUsersV1).Methods("GET")
    v1.HandleFunc("/users/{id}", getUserV1).Methods("GET")
    
    // Version 2
    v2 := r.PathPrefix("/api/v2").Subrouter()
    v2.HandleFunc("/users", getUsersV2).Methods("GET")
    v2.HandleFunc("/users/{id}", getUserV2).Methods("GET")
    
    http.ListenAndServe(":8080", r)
}
```

**Example**:
```bash
# Client using v1
GET /api/v1/users

# Client using v2
GET /api/v2/users
```

**Pros**:
- ✅ Very clear and explicit
- ✅ Easy to route and cache
- ✅ Simple for clients to understand
- ✅ Most widely used in industry
- ✅ Easy to deprecate old versions

**Cons**:
- ❌ Version is part of URL (breaks REST purists)
- ❌ Can lead to URL proliferation

**Best For**: Public APIs, REST APIs, most use cases

---

### Strategy 2: Request Header Versioning

**Pattern**: `API-Version: 2` or `Accept: application/vnd.myapi.v2+json`

```go
func versionMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        version := r.Header.Get("API-Version")
        if version == "" {
            version = "1"  // Default to v1
        }
        
        ctx := context.WithValue(r.Context(), "api-version", version)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func getUsers(w http.ResponseWriter, r *http.Request) {
    version := r.Context().Value("api-version").(string)
    
    switch version {
    case "1":
        getUsersV1(w, r)
    case "2":
        getUsersV2(w, r)
    default:
        respondError(w, http.StatusBadRequest, "Unsupported API version")
    }
}
```

**Example**:
```bash
# Client using v1
curl -H "API-Version: 1" http://api.example.com/users

# Client using v2
curl -H "API-Version: 2" http://api.example.com/users
```

**Pros**:
- ✅ Clean URLs
- ✅ RESTful (version is metadata, not part of resource)
- ✅ Can version independently of URL structure

**Cons**:
- ❌ Less discoverable
- ❌ Harder to cache
- ❌ More complex for clients
- ❌ Can't test in browser easily

**Best For**: Internal APIs, GraphQL-style APIs

---

### Strategy 3: Query Parameter Versioning

**Pattern**: `?version=2`

```go
func getUsers(w http.ResponseWriter, r *http.Request) {
    version := r.URL.Query().Get("version")
    if version == "" {
        version = "1"  // Default
    }
    
    switch version {
    case "1":
        getUsersV1(w, r)
    case "2":
        getUsersV2(w, r)
    default:
        respondError(w, http.StatusBadRequest, "Unsupported version")
    }
}
```

**Example**:
```bash
GET /api/users?version=2
```

**Pros**:
- ✅ Easy to test in browser
- ✅ Simple to implement

**Cons**:
- ❌ Version mixed with query parameters
- ❌ Not commonly used
- ❌ Can interfere with caching

**Best For**: Rarely used, not recommended

---

### Strategy 4: Content Negotiation (Accept Header)

**Pattern**: `Accept: application/vnd.myapi.v2+json`

```go
func contentNegotiationMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        accept := r.Header.Get("Accept")
        
        version := "1"  // Default
        if strings.Contains(accept, "vnd.myapi.v2") {
            version = "2"
        } else if strings.Contains(accept, "vnd.myapi.v1") {
            version = "1"
        }
        
        ctx := context.WithValue(r.Context(), "api-version", version)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

**Example**:
```bash
curl -H "Accept: application/vnd.myapi.v2+json" http://api.example.com/users
```

**Pros**:
- ✅ True RESTful approach
- ✅ Follows HTTP standards

**Cons**:
- ❌ Most complex to implement
- ❌ Hardest for clients to use
- ❌ Not widely adopted

**Best For**: Academic REST APIs, rarely used in practice

---

## 4.4 Implementing URL Path Versioning (Recommended)

### Basic Setup

```go
package main

import (
    "encoding/json"
    "net/http"
    "time"
    
    "github.com/gorilla/mux"
)

// V1 models
type UserV1 struct {
    ID       int    `json:"id"`
    Name     string `json:"name"`
    Email    string `json:"email"`
}

// V2 models - with breaking changes
type UserV2 struct {
    ID        int       `json:"id"`
    FirstName string    `json:"first_name"`  // Split name
    LastName  string    `json:"last_name"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`  // New field
}

// V1 handlers
func getUsersV1(w http.ResponseWriter, r *http.Request) {
    users := []UserV1{
        {ID: 1, Name: "John Doe", Email: "john@example.com"},
        {ID: 2, Name: "Jane Smith", Email: "jane@example.com"},
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func getUserV1(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id := vars["id"]
    
    user := UserV1{
        ID:    1,
        Name:  "John Doe",
        Email: "john@example.com",
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

// V2 handlers
func getUsersV2(w http.ResponseWriter, r *http.Request) {
    users := []UserV2{
        {
            ID:        1,
            FirstName: "John",
            LastName:  "Doe",
            Email:     "john@example.com",
            CreatedAt: time.Now(),
        },
        {
            ID:        2,
            FirstName: "Jane",
            LastName:  "Smith",
            Email:     "jane@example.com",
            CreatedAt: time.Now(),
        },
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func getUserV2(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id := vars["id"]
    
    user := UserV2{
        ID:        1,
        FirstName: "John",
        LastName:  "Doe",
        Email:     "john@example.com",
        CreatedAt: time.Now(),
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

func main() {
    r := mux.NewRouter()
    
    // API V1
    v1 := r.PathPrefix("/api/v1").Subrouter()
    v1.HandleFunc("/users", getUsersV1).Methods("GET")
    v1.HandleFunc("/users/{id}", getUserV1).Methods("GET")
    
    // API V2
    v2 := r.PathPrefix("/api/v2").Subrouter()
    v2.HandleFunc("/users", getUsersV2).Methods("GET")
    v2.HandleFunc("/users/{id}", getUserV2).Methods("GET")
    
    http.ListenAndServe(":8080", r)
}
```

**Test it**:
```bash
# V1 response
curl http://localhost:8080/api/v1/users
# [{"id":1,"name":"John Doe","email":"john@example.com"}]

# V2 response
curl http://localhost:8080/api/v2/users
# [{"id":1,"first_name":"John","last_name":"Doe","email":"john@example.com","created_at":"..."}]
```

---

## 4.5 Code Organization for Multiple Versions

### Recommended Structure

```
myapi/
├── main.go
├── handlers/
│   ├── v1/
│   │   ├── user.go
│   │   └── post.go
│   └── v2/
│       ├── user.go
│       └── post.go
├── models/
│   ├── v1/
│   │   └── user.go
│   └── v2/
│       └── user.go
└── shared/
    ├── middleware/
    └── database/
```

### Example: handlers/v1/user.go

```go
package v1

import (
    "encoding/json"
    "net/http"
    
    modelsv1 "myapi/models/v1"
)

type UserHandler struct {
    // dependencies
}

func (h *UserHandler) GetUsers(w http.ResponseWriter, r *http.Request) {
    users := []modelsv1.User{
        {ID: 1, Name: "John Doe"},
    }
    
    json.NewEncoder(w).Encode(users)
}
```

### Example: handlers/v2/user.go

```go
package v2

import (
    "encoding/json"
    "net/http"
    
    modelsv2 "myapi/models/v2"
)

type UserHandler struct {
    // dependencies
}

func (h *UserHandler) GetUsers(w http.ResponseWriter, r *http.Request) {
    users := []modelsv2.User{
        {ID: 1, FirstName: "John", LastName: "Doe"},
    }
    
    json.NewEncoder(w).Encode(users)
}
```

### Example: main.go

```go
package main

import (
    "net/http"
    
    v1handlers "myapi/handlers/v1"
    v2handlers "myapi/handlers/v2"
    
    "github.com/gorilla/mux"
)

func main() {
    r := mux.NewRouter()
    
    // V1
    v1UserHandler := &v1handlers.UserHandler{}
    v1 := r.PathPrefix("/api/v1").Subrouter()
    v1.HandleFunc("/users", v1UserHandler.GetUsers).Methods("GET")
    
    // V2
    v2UserHandler := &v2handlers.UserHandler{}
    v2 := r.PathPrefix("/api/v2").Subrouter()
    v2.HandleFunc("/users", v2UserHandler.GetUsers).Methods("GET")
    
    http.ListenAndServe(":8080", r)
}
```

---

## 4.6 Sharing Code Between Versions

Not everything needs to be duplicated!

### Share Common Logic

```go
// shared/database/user.go
package database

type UserRecord struct {
    ID        int
    FirstName string
    LastName  string
    Email     string
    CreatedAt time.Time
}

func GetUserByID(id int) (*UserRecord, error) {
    // Database query - shared by all versions
}

func GetAllUsers() ([]UserRecord, error) {
    // Database query - shared by all versions
}
```

### Convert to Version-Specific Models

```go
// handlers/v1/user.go
package v1

import (
    "myapi/shared/database"
    modelsv1 "myapi/models/v1"
)

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    // Get from shared database
    record, err := database.GetUserByID(123)
    if err != nil {
        // Handle error
    }
    
    // Convert to V1 model
    user := modelsv1.User{
        ID:    record.ID,
        Name:  record.FirstName + " " + record.LastName,  // Combine
        Email: record.Email,
    }
    
    json.NewEncoder(w).Encode(user)
}
```

```go
// handlers/v2/user.go
package v2

import (
    "myapi/shared/database"
    modelsv2 "myapi/models/v2"
)

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    // Get from shared database
    record, err := database.GetUserByID(123)
    if err != nil {
        // Handle error
    }
    
    // Convert to V2 model
    user := modelsv2.User{
        ID:        record.ID,
        FirstName: record.FirstName,  // Separate
        LastName:  record.LastName,
        Email:     record.Email,
        CreatedAt: record.CreatedAt,
    }
    
    json.NewEncoder(w).Encode(user)
}
```

---

## 4.7 Deprecation Strategy

### Announce Deprecation

Add headers to deprecated versions:

```go
func deprecationMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Add deprecation headers
        w.Header().Set("Deprecation", "true")
        w.Header().Set("Sunset", "2024-12-31")  // When it will be removed
        w.Header().Set("Link", "</api/v2/users>; rel=\"successor-version\"")
        
        next.ServeHTTP(w, r)
    })
}

func main() {
    r := mux.NewRouter()
    
    // V1 - deprecated
    v1 := r.PathPrefix("/api/v1").Subrouter()
    v1.Use(deprecationMiddleware)  // Add deprecation headers
    v1.HandleFunc("/users", getUsersV1).Methods("GET")
    
    // V2 - current
    v2 := r.PathPrefix("/api/v2").Subrouter()
    v2.HandleFunc("/users", getUsersV2).Methods("GET")
    
    http.ListenAndServe(":8080", r)
}
```

**Response Headers**:
```
Deprecation: true
Sunset: 2024-12-31
Link: </api/v2/users>; rel="successor-version"
```

### Deprecation Timeline

**Best Practice**:

1. **Announcement** (T-6 months): Announce deprecation, add headers
2. **Warning** (T-3 months): Log warnings, send emails to active users
3. **Sunset** (T-0): Remove old version

**Example Timeline**:
```
Jan 1: Launch v2, announce v1 deprecation
Apr 1: Send migration guides to v1 users
Jul 1: Remove v1 endpoints
```

---

## 4.8 Version Detection and Analytics

Track which versions are being used:

```go
func versionAnalyticsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Extract version from path
        if strings.HasPrefix(r.URL.Path, "/api/v1") {
            // Log v1 usage
            log.Printf("V1 usage: %s %s from %s", r.Method, r.URL.Path, r.RemoteAddr)
        } else if strings.HasPrefix(r.URL.Path, "/api/v2") {
            // Log v2 usage
            log.Printf("V2 usage: %s %s from %s", r.Method, r.URL.Path, r.RemoteAddr)
        }
        
        next.ServeHTTP(w, r)
    })
}
```

**Use metrics to**:
- Understand adoption rates
- Identify slow-to-migrate clients
- Make informed decisions about deprecation

---

## 4.9 Migration Strategies

### Strategy 1: Gradual Migration

Provide both versions, encourage migration:

```go
// V1 - returns old format + migration hint
func getUsersV1(w http.ResponseWriter, r *http.Request) {
    users := []UserV1{
        {ID: 1, Name: "John Doe"},
    }
    
    // Add migration hint header
    w.Header().Set("X-API-Upgrade-Available", "v2")
    w.Header().Set("X-API-Upgrade-Guide", "https://docs.api.com/migration-v1-to-v2")
    
    json.NewEncoder(w).Encode(users)
}
```

### Strategy 2: Dual Write

Write to both old and new data structures during transition:

```go
func updateUser(userID int, data UpdateData) error {
    // Write to new format
    err := writeToNewFormat(userID, data)
    if err != nil {
        return err
    }
    
    // Also write to old format (for compatibility)
    err = writeToOldFormat(userID, data)
    if err != nil {
        log.Printf("Warning: old format write failed: %v", err)
        // Don't fail - new format is source of truth
    }
    
    return nil
}
```

### Strategy 3: Adapter Pattern

Make V1 call V2 internally:

```go
// V1 handler wraps V2
func getUsersV1(w http.ResponseWriter, r *http.Request) {
    // Get V2 data
    v2Users := getUsersV2Data()
    
    // Convert to V1 format
    v1Users := make([]UserV1, len(v2Users))
    for i, v2User := range v2Users {
        v1Users[i] = UserV1{
            ID:    v2User.ID,
            Name:  v2User.FirstName + " " + v2User.LastName,
            Email: v2User.Email,
        }
    }
    
    json.NewEncoder(w).Encode(v1Users)
}
```

---

## 4.10 Best Practices

### ✅ DO

1. **Use URL path versioning** (`/api/v1/`) for most APIs
2. **Version from day one** - Don't wait until you need to change something
3. **Make v1 the default** if no version specified (for backward compatibility)
4. **Document breaking changes** clearly
5. **Give advance notice** before deprecating versions
6. **Keep old versions running** for at least 6 months
7. **Share common code** between versions where possible
8. **Use semantic versioning** concepts (major.minor.patch)
9. **Provide migration guides** when releasing new versions

### ❌ DON'T

1. **Don't version unnecessarily** - Only when you have breaking changes
2. **Don't break APIs without versioning** - This crashes client apps
3. **Don't use version numbers > 5** - If you're at v6, something's wrong
4. **Don't deprecate too quickly** - Give clients time to migrate
5. **Don't forget to update documentation** for new versions
6. **Don't duplicate all code** - Share business logic
7. **Don't version every endpoint differently** - Version the entire API

---

## 4.11 Complete Example: Two Versions

```go
package main

import (
    "encoding/json"
    "net/http"
    "time"
    
    "github.com/gorilla/mux"
)

// V1 Models
type UserV1 struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

// V2 Models
type UserV2 struct {
    ID        int       `json:"id"`
    FirstName string    `json:"first_name"`
    LastName  string    `json:"last_name"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
}

// Shared database representation
type UserRecord struct {
    ID        int
    FirstName string
    LastName  string
    Email     string
    CreatedAt time.Time
}

// Simulated database
var users = []UserRecord{
    {ID: 1, FirstName: "John", LastName: "Doe", Email: "john@example.com", CreatedAt: time.Now()},
    {ID: 2, FirstName: "Jane", LastName: "Smith", Email: "jane@example.com", CreatedAt: time.Now()},
}

// V1 Handlers
func getUsersV1(w http.ResponseWriter, r *http.Request) {
    v1Users := make([]UserV1, len(users))
    for i, u := range users {
        v1Users[i] = UserV1{
            ID:    u.ID,
            Name:  u.FirstName + " " + u.LastName,
            Email: u.Email,
        }
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(v1Users)
}

// V2 Handlers
func getUsersV2(w http.ResponseWriter, r *http.Request) {
    v2Users := make([]UserV2, len(users))
    for i, u := range users {
        v2Users[i] = UserV2{
            ID:        u.ID,
            FirstName: u.FirstName,
            LastName:  u.LastName,
            Email:     u.Email,
            CreatedAt: u.CreatedAt,
        }
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(v2Users)
}

// Deprecation middleware
func deprecationMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Deprecation", "true")
        w.Header().Set("Sunset", "2024-12-31")
        w.Header().Set("Link", "</api/v2/users>; rel=\"successor-version\"")
        next.ServeHTTP(w, r)
    })
}

func main() {
    r := mux.NewRouter()
    
    // V1 - Deprecated
    v1 := r.PathPrefix("/api/v1").Subrouter()
    v1.Use(deprecationMiddleware)
    v1.HandleFunc("/users", getUsersV1).Methods("GET")
    
    // V2 - Current
    v2 := r.PathPrefix("/api/v2").Subrouter()
    v2.HandleFunc("/users", getUsersV2).Methods("GET")
    
    http.ListenAndServe(":8080", r)
}
```

---

## Key Takeaways

✅ **URL path versioning** is the most practical approach  
✅ **Version from day one** - Don't wait for breaking changes  
✅ **Breaking changes** require a new version  
✅ **Non-breaking changes** can be added to existing versions  
✅ **Share code** between versions where possible  
✅ **Deprecate gracefully** with advance notice and headers  
✅ **Organize code** by version for clarity  
✅ **Provide migration guides** to help clients upgrade  

---

## What's Next?

Unit 5 will cover API Documentation with Swagger/OpenAPI - automatically generating interactive documentation for all your API versions! 📚
