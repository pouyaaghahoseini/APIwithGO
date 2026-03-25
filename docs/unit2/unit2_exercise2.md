# Unit 2 - Exercise 2: Blog Post API with Categories

**Difficulty**: Intermediate  
**Estimated Time**: 60-75 minutes  
**Concepts Covered**: Related resources, nested routes, advanced filtering, custom middleware

---

## Objective

Build a more complex API with multiple related resources (posts and categories). This exercise introduces working with relationships, nested routes, and more advanced filtering.

---

## Requirements

### Data Models

```go
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
```

### API Endpoints to Implement

#### Categories
| Method | Path | Description |
|--------|------|-------------|
| GET | /categories | List all categories |
| GET | /categories/{id} | Get single category |
| POST | /categories | Create category |
| PUT | /categories/{id} | Update category |
| DELETE | /categories/{id} | Delete category |

#### Posts
| Method | Path | Description |
|--------|------|-------------|
| GET | /posts | List all posts (with filters) |
| GET | /posts/{id} | Get single post with category |
| POST | /posts | Create post |
| PUT | /posts/{id} | Update post |
| DELETE | /posts/{id} | Delete post |

#### Nested Routes
| Method | Path | Description |
|--------|------|-------------|
| GET | /categories/{id}/posts | Get all posts in a category |

### Query Parameters

**GET /posts** should support:
- `?category=technology` - Filter by category slug
- `?author=john` - Filter by author
- `?published=true` - Filter by published status
- `?search=golang` - Search in title and content
- Multiple filters can be combined

### Validation Requirements

**Category**:
- Name must not be empty (min 2 characters)
- Slug must be lowercase, alphanumeric, and dashes only
- Slug must be unique

**Post**:
- Title must not be empty (min 5 characters)
- Content must not be empty (min 10 characters)
- Author must not be empty
- CategoryID must reference an existing category

### Error Responses

```json
{
  "error": "validation_failed",
  "message": "Title must be at least 5 characters"
}

{
  "error": "not_found",
  "message": "Category not found"
}

{
  "error": "conflict",
  "message": "Category slug already exists"
}
```

### Custom Middleware Requirements

1. **Request ID Middleware**: Add unique request ID to each request
2. **Logging Middleware**: Log with request ID
3. **Recovery Middleware**: Catch panics and return 500
4. **Rate Limit Middleware**: Simple rate limiting (max 100 requests/minute per IP)

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
    postsMu      sync.RWMutex
)

// TODO: Implement category handlers
func getCategories(w http.ResponseWriter, r *http.Request) {}
func getCategory(w http.ResponseWriter, r *http.Request) {}
func createCategory(w http.ResponseWriter, r *http.Request) {}
func updateCategory(w http.ResponseWriter, r *http.Request) {}
func deleteCategory(w http.ResponseWriter, r *http.Request) {}

// TODO: Implement post handlers
func getPosts(w http.ResponseWriter, r *http.Request) {}
func getPost(w http.ResponseWriter, r *http.Request) {}
func createPost(w http.ResponseWriter, r *http.Request) {}
func updatePost(w http.ResponseWriter, r *http.Request) {}
func deletePost(w http.ResponseWriter, r *http.Request) {}

// TODO: Implement nested route handler
func getCategoryPosts(w http.ResponseWriter, r *http.Request) {}

// TODO: Implement middleware
func requestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Add unique request ID
    })
}

func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Log with request ID
    })
}

func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Recover from panics
    })
}

func rateLimitMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Simple rate limiting
    })
}

// Helper functions
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, err, message string) {
    respondJSON(w, status, ErrorResponse{Error: err, Message: message})
}

func main() {
    r := mux.NewRouter()
    
    // TODO: Register routes and middleware
    
    fmt.Println("Server starting on http://localhost:8080")
    http.ListenAndServe(":8080", r)
}
```

---

## Testing Your API

### Setup: Create Categories
```bash
# Create Technology category
curl -X POST http://localhost:8080/categories \
  -H "Content-Type: application/json" \
  -d '{"name":"Technology","slug":"technology","description":"Tech articles"}'

# Create Programming category
curl -X POST http://localhost:8080/categories \
  -H "Content-Type: application/json" \
  -d '{"name":"Programming","slug":"programming","description":"Programming tutorials"}'

# Create Lifestyle category
curl -X POST http://localhost:8080/categories \
  -H "Content-Type: application/json" \
  -d '{"name":"Lifestyle","slug":"lifestyle","description":"Life tips"}'
```

### Create Posts
```bash
# Create post in Technology category
curl -X POST http://localhost:8080/posts \
  -H "Content-Type: application/json" \
  -d '{
    "title":"Introduction to Go Programming",
    "content":"Go is a statically typed, compiled programming language...",
    "author":"John Doe",
    "category_id":1,
    "published":true
  }'

# Create post in Programming category
curl -X POST http://localhost:8080/posts \
  -H "Content-Type: application/json" \
  -d '{
    "title":"Building REST APIs with Go",
    "content":"REST APIs are essential for modern web development...",
    "author":"Jane Smith",
    "category_id":2,
    "published":true
  }'

# Create unpublished draft
curl -X POST http://localhost:8080/posts \
  -H "Content-Type: application/json" \
  -d '{
    "title":"Advanced Go Patterns",
    "content":"This post covers advanced patterns in Go...",
    "author":"John Doe",
    "category_id":2,
    "published":false
  }'
```

### Test Filtering
```bash
# Get all posts
curl http://localhost:8080/posts

# Filter by category
curl "http://localhost:8080/posts?category=technology"

# Filter by author
curl "http://localhost:8080/posts?author=John%20Doe"

# Filter by published status
curl "http://localhost:8080/posts?published=true"

# Search in title/content
curl "http://localhost:8080/posts?search=REST"

# Combine filters
curl "http://localhost:8080/posts?category=programming&author=Jane%20Smith"
```

### Test Nested Routes
```bash
# Get all posts in Technology category
curl http://localhost:8080/categories/1/posts
```

### Test Validation
```bash
# Should fail - title too short
curl -X POST http://localhost:8080/posts \
  -H "Content-Type: application/json" \
  -d '{"title":"Go","content":"Short content","author":"Test","category_id":1}'

# Should fail - invalid category
curl -X POST http://localhost:8080/posts \
  -H "Content-Type: application/json" \
  -d '{"title":"Valid Title","content":"Valid content","author":"Test","category_id":999}'

# Should fail - duplicate slug
curl -X POST http://localhost:8080/categories \
  -H "Content-Type: application/json" \
  -d '{"name":"Tech","slug":"technology","description":"Duplicate slug"}'
```

### Test Cascade Delete
```bash
# Try deleting category with posts (should fail or cascade)
curl -X DELETE http://localhost:8080/categories/1
```

---

## Expected Outputs

### GET /posts (with category embedded)
```json
[
  {
    "id": 1,
    "title": "Introduction to Go Programming",
    "content": "Go is a statically typed...",
    "author": "John Doe",
    "category_id": 1,
    "published": true,
    "created_at": "2024-01-15T10:30:00Z",
    "updated_at": "2024-01-15T10:30:00Z",
    "category": {
      "id": 1,
      "name": "Technology",
      "slug": "technology",
      "description": "Tech articles",
      "created_at": "2024-01-15T10:25:00Z"
    }
  }
]
```

### GET /categories/1/posts
```json
[
  {
    "id": 1,
    "title": "Introduction to Go Programming",
    "content": "Go is a statically typed...",
    "author": "John Doe",
    "category_id": 1,
    "published": true,
    "created_at": "2024-01-15T10:30:00Z",
    "updated_at": "2024-01-15T10:30:00Z",
    "category": {
      "id": 1,
      "name": "Technology",
      "slug": "technology",
      "description": "Tech articles",
      "created_at": "2024-01-15T10:25:00Z"
    }
  }
]
```

### Console Output (with Request ID)
```
[2024-01-15 10:30:00] [req-abc123] POST /posts - 201 - 5ms
[2024-01-15 10:30:05] [req-def456] GET /posts?category=technology - 200 - 2ms
[2024-01-15 10:30:10] [req-ghi789] GET /categories/1/posts - 200 - 3ms
```

---

## Bonus Challenges

### Bonus 1: Pagination
Add pagination to /posts and /categories:
```bash
curl "http://localhost:8080/posts?page=1&limit=10"
```

Response should include pagination metadata:
```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 45,
    "pages": 5
  }
}
```

### Bonus 2: Post Count in Categories
Add post count when listing categories:
```json
{
  "id": 1,
  "name": "Technology",
  "slug": "technology",
  "description": "Tech articles",
  "post_count": 5,
  "created_at": "2024-01-15T10:25:00Z"
}
```

### Bonus 3: Prevent Deleting Categories with Posts
Return 409 Conflict if trying to delete a category that has posts:
```json
{
  "error": "conflict",
  "message": "Cannot delete category with existing posts"
}
```

### Bonus 4: Bulk Operations
Add endpoint to publish/unpublish multiple posts:
```bash
curl -X PATCH http://localhost:8080/posts/bulk \
  -H "Content-Type: application/json" \
  -d '{"ids":[1,2,3],"published":true}'
```

### Bonus 5: Recent Posts Endpoint
Add GET /posts/recent that returns the 5 most recent published posts:
```bash
curl http://localhost:8080/posts/recent
```

---

## Hints

### Hint 1: Slug Validation
```go
import "regexp"

func isValidSlug(slug string) bool {
    // Only lowercase letters, numbers, and dashes
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
```

### Hint 2: Getting Post with Category
```go
func getPostWithCategory(postID int) (*PostWithCategory, error) {
    postsMu.RLock()
    post, exists := posts[postID]
    postsMu.RUnlock()
    
    if !exists {
        return nil, errors.New("post not found")
    }
    
    categoriesMu.RLock()
    category, exists := categories[post.CategoryID]
    categoriesMu.RUnlock()
    
    if !exists {
        return nil, errors.New("category not found")
    }
    
    return &PostWithCategory{
        Post:     post,
        Category: category,
    }, nil
}
```

### Hint 3: Advanced Filtering
```go
func getPosts(w http.ResponseWriter, r *http.Request) {
    categorySlug := r.URL.Query().Get("category")
    author := r.URL.Query().Get("author")
    publishedParam := r.URL.Query().Get("published")
    search := r.URL.Query().Get("search")
    
    postsMu.RLock()
    categoriesMu.RLock()
    defer postsMu.RUnlock()
    defer categoriesMu.RUnlock()
    
    var results []PostWithCategory
    
    for _, post := range posts {
        // Filter by category slug
        if categorySlug != "" {
            category := categories[post.CategoryID]
            if category.Slug != categorySlug {
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
            searchLower := strings.ToLower(search)
            titleMatch := strings.Contains(strings.ToLower(post.Title), searchLower)
            contentMatch := strings.Contains(strings.ToLower(post.Content), searchLower)
            if !titleMatch && !contentMatch {
                continue
            }
        }
        
        // Add to results
        category := categories[post.CategoryID]
        results = append(results, PostWithCategory{
            Post:     post,
            Category: category,
        })
    }
    
    respondJSON(w, http.StatusOK, results)
}
```

### Hint 4: Request ID Middleware
```go
import (
    "context"
    "github.com/google/uuid"
)

type contextKey string

const requestIDKey contextKey = "requestID"

func requestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Generate unique ID
        requestID := uuid.New().String()
        
        // Add to context
        ctx := context.WithValue(r.Context(), requestIDKey, requestID)
        
        // Add to response header
        w.Header().Set("X-Request-ID", requestID)
        
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func getRequestID(r *http.Request) string {
    if id, ok := r.Context().Value(requestIDKey).(string); ok {
        return id
    }
    return "unknown"
}
```

### Hint 5: Simple Rate Limiting
```go
type visitor struct {
    count    int
    lastSeen time.Time
}

var (
    visitors = make(map[string]*visitor)
    mu       sync.Mutex
)

func rateLimitMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ip := r.RemoteAddr
        
        mu.Lock()
        v, exists := visitors[ip]
        now := time.Now()
        
        if !exists {
            visitors[ip] = &visitor{count: 1, lastSeen: now}
            mu.Unlock()
            next.ServeHTTP(w, r)
            return
        }
        
        // Reset if window passed
        if now.Sub(v.lastSeen) > time.Minute {
            v.count = 1
            v.lastSeen = now
            mu.Unlock()
            next.ServeHTTP(w, r)
            return
        }
        
        if v.count >= 100 {
            mu.Unlock()
            respondError(w, http.StatusTooManyRequests, 
                "rate_limit_exceeded", 
                "Too many requests. Please try again later.")
            return
        }
        
        v.count++
        v.lastSeen = now
        mu.Unlock()
        
        next.ServeHTTP(w, r)
    })
}
```

---

## What You're Learning

✅ Working with related resources  
✅ Nested routes and resource relationships  
✅ Advanced filtering with multiple parameters  
✅ Request ID tracking  
✅ Recovery from panics  
✅ Simple rate limiting  
✅ Slug validation and uniqueness  
✅ Complex validation rules  
✅ Cascade delete considerations  

This exercise demonstrates real-world API patterns with multiple related resources!
