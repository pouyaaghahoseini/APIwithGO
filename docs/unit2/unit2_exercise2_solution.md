# Unit 2 - Exercise 2 Solution: Blog Post API with Categories

**Complete implementation with explanations**

---

## Full Solution Code

```go
package main

import (
    "context"
    "encoding/json"
    "errors"
    "fmt"
    "net/http"
    "regexp"
    "strconv"
    "strings"
    "sync"
    "time"

    "github.com/google/uuid"
    "github.com/gorilla/mux"
)

// Models
type Category struct {
    ID          int       `json:"id"`
    Name        string    `json:"name"`
    Slug        string    `json:"slug"`
    Description string    `json:"description"`
    CreatedAt   time.Time `json:"created_at"`
}

type Post struct {
    ID         int       `json:"id"`
    Title      string    `json:"title"`
    Content    string    `json:"content"`
    Author     string    `json:"author"`
    CategoryID int       `json:"category_id"`
    Published  bool      `json:"published"`
    CreatedAt  time.Time `json:"created_at"`
    UpdatedAt  time.Time `json:"updated_at"`
}

type PostWithCategory struct {
    Post
    Category Category `json:"category"`
}

type ErrorResponse struct {
    Error   string `json:"error"`
    Message string `json:"message"`
}

// Storage
var (
    categories   = make(map[int]Category)
    posts        = make(map[int]Post)
    nextCatID    = 1
    nextPostID   = 1
    categoriesMu sync.RWMutex
    postsMu      sync.RWMutex)

// Context keys
type contextKey string

const requestIDKey contextKey = "requestID"

// =============================================================================
// CATEGORY HANDLERS
// =============================================================================

func getCategories(w http.ResponseWriter, r *http.Request) {
    categoriesMu.RLock()
    defer categoriesMu.RUnlock()

    categoryList := make([]Category, 0, len(categories))
    for _, cat := range categories {
        categoryList = append(categoryList, cat)
    }

    respondJSON(w, http.StatusOK, categoryList)
}

func getCategory(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        respondError(w, http.StatusBadRequest, "validation_failed", "Invalid category ID")
        return
    }

    categoriesMu.RLock()
    category, exists := categories[id]
    categoriesMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "not_found", "Category not found")
        return
    }

    respondJSON(w, http.StatusOK, category)
}

func createCategory(w http.ResponseWriter, r *http.Request) {
    var category Category
    err := json.NewDecoder(r.Body).Decode(&category)
    if err != nil {
        respondError(w, http.StatusBadRequest, "invalid_json", "Invalid JSON format")
        return
    }
    defer r.Body.Close()

    // Validate
    if err := validateCategory(&category, 0); err != nil {
        respondError(w, http.StatusBadRequest, "validation_failed", err.Error())
        return
    }

    // Check slug uniqueness
    if slugExists(category.Slug, 0) {
        respondError(w, http.StatusConflict, "conflict", "Category slug already exists")
        return
    }

    // Create
    categoriesMu.Lock()
    category.ID = nextCatID
    category.CreatedAt = time.Now()
    categories[nextCatID] = category
    nextCatID++
    categoriesMu.Unlock()

    respondJSON(w, http.StatusCreated, category)
}

func updateCategory(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        respondError(w, http.StatusBadRequest, "validation_failed", "Invalid category ID")
        return
    }

    var category Category
    err = json.NewDecoder(r.Body).Decode(&category)
    if err != nil {
        respondError(w, http.StatusBadRequest, "invalid_json", "Invalid JSON format")
        return
    }
    defer r.Body.Close()

    // Validate
    if err := validateCategory(&category, id); err != nil {
        respondError(w, http.StatusBadRequest, "validation_failed", err.Error())
        return
    }

    // Check slug uniqueness (excluding current category)
    if slugExists(category.Slug, id) {
        respondError(w, http.StatusConflict, "conflict", "Category slug already exists")
        return
    }

    // Update
    categoriesMu.Lock()
    existing, exists := categories[id]
    if !exists {
        categoriesMu.Unlock()
        respondError(w, http.StatusNotFound, "not_found", "Category not found")
        return
    }

    category.ID = id
    category.CreatedAt = existing.CreatedAt
    categories[id] = category
    categoriesMu.Unlock()

    respondJSON(w, http.StatusOK, category)
}

func deleteCategory(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        respondError(w, http.StatusBadRequest, "validation_failed", "Invalid category ID")
        return
    }

    // Check if category has posts
    postsMu.RLock()
    hasPost := false
    for _, post := range posts {
        if post.CategoryID == id {
            hasPost = true
            break
        }
    }
    postsMu.RUnlock()

    if hasPost {
        respondError(w, http.StatusConflict, "conflict",
            "Cannot delete category with existing posts")
        return
    }

    categoriesMu.Lock()
    _, exists := categories[id]
    if !exists {
        categoriesMu.Unlock()
        respondError(w, http.StatusNotFound, "not_found", "Category not found")
        return
    }

    delete(categories, id)
    categoriesMu.Unlock()

    w.WriteHeader(http.StatusNoContent)
}

// =============================================================================
// POST HANDLERS
// =============================================================================

func getPosts(w http.ResponseWriter, r *http.Request) {
    categorySlug := r.URL.Query().Get("category")
    author := r.URL.Query().Get("author")
    publishedParam := r.URL.Query().Get("published")
    search := strings.ToLower(r.URL.Query().Get("search"))

    postsMu.RLock()
    categoriesMu.RLock()
    defer postsMu.RUnlock()
    defer categoriesMu.RUnlock()

    var results []PostWithCategory

    for _, post := range posts {
        // Filter by category slug
        if categorySlug != "" {
            category, exists := categories[post.CategoryID]
            if !exists || category.Slug != categorySlug {
                continue
            }
        }

        // Filter by author
        if author != "" && post.Author != author {
            continue
        }

        // Filter by published status
        if publishedParam != "" {
            isPublished := publishedParam == "true"
            if post.Published != isPublished {
                continue
            }
        }

        // Search in title and content
        if search != "" {
            titleMatch := strings.Contains(strings.ToLower(post.Title), search)
            contentMatch := strings.Contains(strings.ToLower(post.Content), search)
            if !titleMatch && !contentMatch {
                continue
            }
        }

        // Add to results with category
        category := categories[post.CategoryID]
        results = append(results, PostWithCategory{
            Post:     post,
            Category: category,
        })
    }

    respondJSON(w, http.StatusOK, results)
}

func getPost(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        respondError(w, http.StatusBadRequest, "validation_failed", "Invalid post ID")
        return
    }

    postsMu.RLock()
    post, exists := posts[id]
    postsMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "not_found", "Post not found")
        return
    }

    categoriesMu.RLock()
    category, exists := categories[post.CategoryID]
    categoriesMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "not_found", "Category not found")
        return
    }

    result := PostWithCategory{
        Post:     post,
        Category: category,
    }

    respondJSON(w, http.StatusOK, result)
}

func createPost(w http.ResponseWriter, r *http.Request) {
    var post Post
    err := json.NewDecoder(r.Body).Decode(&post)
    if err != nil {
        respondError(w, http.StatusBadRequest, "invalid_json", "Invalid JSON format")
        return
    }
    defer r.Body.Close()

    // Validate
    if err := validatePost(&post); err != nil {
        respondError(w, http.StatusBadRequest, "validation_failed", err.Error())
        return
    }

    // Check category exists
    categoriesMu.RLock()
    category, categoryExists := categories[post.CategoryID]
    categoriesMu.RUnlock()

    if !categoryExists {
        respondError(w, http.StatusBadRequest, "validation_failed",
            "Category does not exist")
        return
    }

    // Create
    postsMu.Lock()
    post.ID = nextPostID
    post.CreatedAt = time.Now()
    post.UpdatedAt = time.Now()
    posts[nextPostID] = post
    nextPostID++
    postsMu.Unlock()

    result := PostWithCategory{
        Post:     post,
        Category: category,
    }

    respondJSON(w, http.StatusCreated, result)
}

func updatePost(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        respondError(w, http.StatusBadRequest, "validation_failed", "Invalid post ID")
        return
    }

    var post Post
    err = json.NewDecoder(r.Body).Decode(&post)
    if err != nil {
        respondError(w, http.StatusBadRequest, "invalid_json", "Invalid JSON format")
        return
    }
    defer r.Body.Close()

    // Validate
    if err := validatePost(&post); err != nil {
        respondError(w, http.StatusBadRequest, "validation_failed", err.Error())
        return
    }

    // Check category exists
    categoriesMu.RLock()
    category, categoryExists := categories[post.CategoryID]
    categoriesMu.RUnlock()

    if !categoryExists {
        respondError(w, http.StatusBadRequest, "validation_failed",
            "Category does not exist")
        return
    }

    // Update
    postsMu.Lock()
    existing, exists := posts[id]
    if !exists {
        postsMu.Unlock()
        respondError(w, http.StatusNotFound, "not_found", "Post not found")
        return
    }

    post.ID = id
    post.CreatedAt = existing.CreatedAt
    post.UpdatedAt = time.Now()
    posts[id] = post
    postsMu.Unlock()

    result := PostWithCategory{
        Post:     post,
        Category: category,
    }

    respondJSON(w, http.StatusOK, result)
}

func deletePost(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        respondError(w, http.StatusBadRequest, "validation_failed", "Invalid post ID")
        return
    }

    postsMu.Lock()
    _, exists := posts[id]
    if !exists {
        postsMu.Unlock()
        respondError(w, http.StatusNotFound, "not_found", "Post not found")
        return
    }

    delete(posts, id)
    postsMu.Unlock()

    w.WriteHeader(http.StatusNoContent)
}

// =============================================================================
// NESTED ROUTE HANDLER
// =============================================================================

func getCategoryPosts(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    categoryID, err := strconv.Atoi(vars["id"])
    if err != nil {
        respondError(w, http.StatusBadRequest, "validation_failed", "Invalid category ID")
        return
    }

    categoriesMu.RLock()
    category, exists := categories[categoryID]
    categoriesMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "not_found", "Category not found")
        return
    }

    postsMu.RLock()
    defer postsMu.RUnlock()

    var results []PostWithCategory
    for _, post := range posts {
        if post.CategoryID == categoryID {
            results = append(results, PostWithCategory{
                Post:     post,
                Category: category,
            })
        }
    }

    respondJSON(w, http.StatusOK, results)
}

// =============================================================================
// VALIDATION
// =============================================================================

func validateCategory(cat *Category, excludeID int) error {
    cat.Name = strings.TrimSpace(cat.Name)
    cat.Slug = strings.TrimSpace(cat.Slug)

    if len(cat.Name) < 2 {
        return errors.New("Category name must be at least 2 characters")
    }

    if cat.Slug == "" {
        return errors.New("Category slug is required")
    }

    if !isValidSlug(cat.Slug) {
        return errors.New("Slug must be lowercase alphanumeric with dashes only")
    }

    return nil
}

func validatePost(post *Post) error {
    post.Title = strings.TrimSpace(post.Title)
    post.Content = strings.TrimSpace(post.Content)
    post.Author = strings.TrimSpace(post.Author)

    if len(post.Title) < 5 {
        return errors.New("Title must be at least 5 characters")
    }

    if len(post.Content) < 10 {
        return errors.New("Content must be at least 10 characters")
    }

    if post.Author == "" {
        return errors.New("Author is required")
    }

    if post.CategoryID <= 0 {
        return errors.New("Valid category ID is required")
    }

    return nil
}

func isValidSlug(slug string) bool {
    match, _ := regexp.MatchString(`^[a-z0-9-]+$`, slug)
    return match
}

func slugExists(slug string, excludeID int) bool {
    categoriesMu.RLock()
    defer categoriesMu.RUnlock()

    for _, cat := range categories {
        if cat.Slug == slug && cat.ID != excludeID {
            return true
        }
    }
    return false
}

// =============================================================================
// MIDDLEWARE
// =============================================================================

func requestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := uuid.New().String()
        ctx := context.WithValue(r.Context(), requestIDKey, requestID)
        w.Header().Set("X-Request-ID", requestID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        requestID := getRequestID(r)

        recorder := &statusRecorder{
            ResponseWriter: w,
            status:         http.StatusOK,
        }

        next.ServeHTTP(recorder, r)

        duration := time.Since(start)
        fmt.Printf("[%s] [%s] %s %s - %d - %v\n",
            time.Now().Format("2006-01-02 15:04:05"),
            requestID,
            r.Method,
            r.URL.Path,
            recorder.status,
            duration)
    })
}

func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                requestID := getRequestID(r)
                fmt.Printf("[PANIC] [%s] %v\n", requestID, err)
                respondError(w, http.StatusInternalServerError,
                    "internal_error", "Internal server error")
            }
        }()
        next.ServeHTTP(w, r)
    })
}

type visitor struct {
    count    int
    lastSeen time.Time
}

var (
    visitors = make(map[string]*visitor)
    visMu    sync.Mutex
)

func rateLimitMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ip := r.RemoteAddr

        visMu.Lock()
        v, exists := visitors[ip]
        now := time.Now()

        if !exists {
            visitors[ip] = &visitor{count: 1, lastSeen: now}
            visMu.Unlock()
            next.ServeHTTP(w, r)
            return
        }

        // Reset if window passed
        if now.Sub(v.lastSeen) > time.Minute {
            v.count = 1
            v.lastSeen = now
            visMu.Unlock()
            next.ServeHTTP(w, r)
            return
        }

        if v.count >= 100 {
            visMu.Unlock()
            respondError(w, http.StatusTooManyRequests,
                "rate_limit_exceeded",
                "Too many requests. Please try again later.")
            return
        }

        v.count++
        v.lastSeen = now
        visMu.Unlock()

        next.ServeHTTP(w, r)
    })
}

type statusRecorder struct {
    http.ResponseWriter
    status int
}

func (r *statusRecorder) WriteHeader(status int) {
    r.status = status
    r.ResponseWriter.WriteHeader(status)
}

// =============================================================================
// HELPERS
// =============================================================================

func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, err, message string) {
    respondJSON(w, status, ErrorResponse{Error: err, Message: message})
}

func getRequestID(r *http.Request) string {
    if id, ok := r.Context().Value(requestIDKey).(string); ok {
        return id
    }
    return "unknown"
}

// =============================================================================
// MAIN
// =============================================================================

func main() {
    r := mux.NewRouter()

    // Global middleware
    r.Use(recoveryMiddleware)
    r.Use(requestIDMiddleware)
    r.Use(loggingMiddleware)
    r.Use(rateLimitMiddleware)

    // Category routes
    r.HandleFunc("/categories", getCategories).Methods("GET")
    r.HandleFunc("/categories/{id}", getCategory).Methods("GET")
    r.HandleFunc("/categories", createCategory).Methods("POST")
    r.HandleFunc("/categories/{id}", updateCategory).Methods("PUT")
    r.HandleFunc("/categories/{id}", deleteCategory).Methods("DELETE")

    // Nested route - must come before /posts/{id}
    r.HandleFunc("/categories/{id}/posts", getCategoryPosts).Methods("GET")

    // Post routes
    r.HandleFunc("/posts", getPosts).Methods("GET")
    r.HandleFunc("/posts/{id}", getPost).Methods("GET")
    r.HandleFunc("/posts", createPost).Methods("POST")
    r.HandleFunc("/posts/{id}", updatePost).Methods("PUT")
    r.HandleFunc("/posts/{id}", deletePost).Methods("DELETE")

    fmt.Println("Server starting on http://localhost:8080")
    http.ListenAndServe(":8080", r)
}
```

---

## Key Concepts Explained

### 1. Request ID with Context

```go
// Store in context
ctx := context.WithValue(r.Context(), requestIDKey, requestID)
next.ServeHTTP(w, r.WithContext(ctx))

// Retrieve from context
func getRequestID(r *http.Request) string {
    if id, ok := r.Context().Value(requestIDKey).(string); ok {
        return id
    }
    return "unknown"
}
```

**Why context?** Request-scoped data that flows through middleware chain without global state.

### 2. Panic Recovery

```go
defer func() {
    if err := recover(); err != nil {
        // Handle panic
        fmt.Printf("[PANIC] %v\n", err)
        respondError(w, 500, "internal_error", "Internal server error")
    }
}()
```

**Critical for production** - prevents one request panic from crashing entire server.

### 3. Slug Validation with Regex

```go
func isValidSlug(slug string) bool {
    match, _ := regexp.MatchString(`^[a-z0-9-]+$`, slug)
    return match
}
```

Pattern `^[a-z0-9-]+$` means:
- `^` - start of string
- `[a-z0-9-]+` - one or more lowercase letters, digits, or dashes
- `$` - end of string

### 4. Relationship Validation

```go
// Check if category exists before creating post
categoriesMu.RLock()
_, exists := categories[post.CategoryID]
categoriesMu.RUnlock()

if !exists {
    return error
}
```

### 5. Preventing Cascade Delete Issues

```go
// Check if category has posts before deleting
postsMu.RLock()
hasPost := false
for _, post := range posts {
    if post.CategoryID == id {
        hasPost = true
        break
    }
}
postsMu.RUnlock()

if hasPost {
    return conflict error
}
```

This complete solution demonstrates production-ready patterns for complex APIs with related resources!
