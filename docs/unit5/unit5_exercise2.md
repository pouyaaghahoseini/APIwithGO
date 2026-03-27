# Unit 5 - Exercise 2: Document Multi-Version Blog API

**Difficulty**: Advanced  
**Estimated Time**: 60-75 minutes  
**Concepts Covered**: Multi-version docs, nested objects, complex schemas, enums, file uploads

---

## Objective

Document a blog API with two versions (v1 and v2) that includes:
- Posts with authors and comments
- Nested objects and relationships
- File upload endpoints
- Different response structures per version
- Complex filtering and pagination

This demonstrates real-world API documentation complexity.

---

## Requirements

### API Structure

The API has two versions with different response formats:

**V1 Structure** (Simple):
- Posts with basic info
- Author as string (username only)
- Comments separate

**V2 Structure** (Enhanced):
- Posts with full details
- Author as object (full info)
- Comments embedded
- Media attachments
- Statistics

### Endpoints to Document

| Version | Method | Path | Description |
|---------|--------|------|-------------|
| Both | GET | /api/v1/posts | List posts (V1 format) |
| Both | GET | /api/v2/posts | List posts (V2 format) |
| Both | GET | /api/v1/posts/{id} | Get post (V1) |
| Both | GET | /api/v2/posts/{id} | Get post (V2) |
| Both | POST | /api/v1/posts | Create post (V1) |
| Both | POST | /api/v2/posts | Create post with media (V2) |
| V2 | POST | /api/v2/posts/{id}/media | Upload media |
| Both | POST | /api/v1/posts/{id}/comments | Add comment (V1) |
| Both | POST | /api/v2/posts/{id}/comments | Add comment (V2) |
| V2 | GET | /api/v2/posts/{id}/stats | Get post statistics |

---

## Models to Document

### V1 Models

```go
type PostV1 struct {
    ID        int       `json:"id"`
    Title     string    `json:"title"`
    Content   string    `json:"content"`
    Author    string    `json:"author"`
    Status    string    `json:"status"`
    CreatedAt time.Time `json:"created_at"`
}

type CommentV1 struct {
    ID        int       `json:"id"`
    PostID    int       `json:"post_id"`
    Author    string    `json:"author"`
    Content   string    `json:"content"`
    CreatedAt time.Time `json:"created_at"`
}

type CreatePostV1Request struct {
    Title   string `json:"title"`
    Content string `json:"content"`
}
```

### V2 Models

```go
type PostV2 struct {
    ID           int           `json:"id"`
    Title        string        `json:"title"`
    Content      string        `json:"content"`
    Excerpt      string        `json:"excerpt"`
    Author       Author        `json:"author"`
    Status       PostStatus    `json:"status"`
    Tags         []string      `json:"tags"`
    Media        []MediaFile   `json:"media"`
    Comments     []CommentV2   `json:"comments"`
    Stats        PostStats     `json:"stats"`
    CreatedAt    time.Time     `json:"created_at"`
    UpdatedAt    time.Time     `json:"updated_at"`
    PublishedAt  *time.Time    `json:"published_at"`
}

type Author struct {
    ID          int    `json:"id"`
    Username    string `json:"username"`
    DisplayName string `json:"display_name"`
    Email       string `json:"email"`
    AvatarURL   string `json:"avatar_url"`
    Bio         string `json:"bio"`
}

type MediaFile struct {
    ID       int    `json:"id"`
    Type     string `json:"type"`
    URL      string `json:"url"`
    Filename string `json:"filename"`
    Size     int64  `json:"size"`
}

type CommentV2 struct {
    ID        int       `json:"id"`
    PostID    int       `json:"post_id"`
    Author    Author    `json:"author"`
    Content   string    `json:"content"`
    Likes     int       `json:"likes"`
    CreatedAt time.Time `json:"created_at"`
}

type PostStats struct {
    Views     int `json:"views"`
    Likes     int `json:"likes"`
    Shares    int `json:"shares"`
    Comments  int `json:"comments"`
}

type PostStatus string

const (
    StatusDraft     PostStatus = "draft"
    StatusPublished PostStatus = "published"
    StatusArchived  PostStatus = "archived"
)

type CreatePostV2Request struct {
    Title   string     `json:"title"`
    Content string     `json:"content"`
    Excerpt string     `json:"excerpt"`
    Tags    []string   `json:"tags"`
    Status  PostStatus `json:"status"`
}
```

### Query Parameters

**GET /posts** (both versions):
- `status` (string): Filter by status (enum: draft, published, archived)
- `author` (string): Filter by author username
- `tag` (string): Filter by tag (V2 only)
- `search` (string): Full-text search
- `sort` (string): Sort by (enum: created_at, updated_at, title, views)
- `order` (string): Sort order (enum: asc, desc)
- `page` (int): Page number (default: 1)
- `limit` (int): Items per page (default: 10, max: 100)

---

## Starter Code

```go
package main

import (
    "encoding/json"
    "net/http"
    "time"

    "github.com/gorilla/mux"
    httpSwagger "github.com/swaggo/http-swagger"

    _ "myapi/docs"
)

// TODO: Add general API info for V1
// TODO: Add general API info for V2

type PostStatus string

const (
    StatusDraft     PostStatus = "draft"
    StatusPublished PostStatus = "published"
    StatusArchived  PostStatus = "archived"
)

// V1 Models - TODO: Add example tags
type PostV1 struct {
    ID        int       `json:"id"`
    Title     string    `json:"title"`
    Content   string    `json:"content"`
    Author    string    `json:"author"`
    Status    string    `json:"status"`
    CreatedAt time.Time `json:"created_at"`
}

type CommentV1 struct {
    ID        int       `json:"id"`
    PostID    int       `json:"post_id"`
    Author    string    `json:"author"`
    Content   string    `json:"content"`
    CreatedAt time.Time `json:"created_at"`
}

type CreatePostV1Request struct {
    Title   string `json:"title" binding:"required"`
    Content string `json:"content" binding:"required"`
}

// V2 Models - TODO: Add example tags
type PostV2 struct {
    ID          int         `json:"id"`
    Title       string      `json:"title"`
    Content     string      `json:"content"`
    Excerpt     string      `json:"excerpt"`
    Author      Author      `json:"author"`
    Status      PostStatus  `json:"status"`
    Tags        []string    `json:"tags"`
    Media       []MediaFile `json:"media"`
    Comments    []CommentV2 `json:"comments"`
    Stats       PostStats   `json:"stats"`
    CreatedAt   time.Time   `json:"created_at"`
    UpdatedAt   time.Time   `json:"updated_at"`
    PublishedAt *time.Time  `json:"published_at"`
}

type Author struct {
    ID          int    `json:"id"`
    Username    string `json:"username"`
    DisplayName string `json:"display_name"`
    Email       string `json:"email"`
    AvatarURL   string `json:"avatar_url"`
    Bio         string `json:"bio"`
}

type MediaFile struct {
    ID       int    `json:"id"`
    Type     string `json:"type"`
    URL      string `json:"url"`
    Filename string `json:"filename"`
    Size     int64  `json:"size"`
}

type CommentV2 struct {
    ID        int       `json:"id"`
    PostID    int       `json:"post_id"`
    Author    Author    `json:"author"`
    Content   string    `json:"content"`
    Likes     int       `json:"likes"`
    CreatedAt time.Time `json:"created_at"`
}

type PostStats struct {
    Views    int `json:"views"`
    Likes    int `json:"likes"`
    Shares   int `json:"shares"`
    Comments int `json:"comments"`
}

type CreatePostV2Request struct {
    Title   string     `json:"title" binding:"required"`
    Content string     `json:"content" binding:"required"`
    Excerpt string     `json:"excerpt"`
    Tags    []string   `json:"tags"`
    Status  PostStatus `json:"status"`
}

type ErrorResponse struct {
    Error string `json:"error"`
}

// TODO: Document V1 handlers
func getPostsV1(w http.ResponseWriter, r *http.Request) {}
func getPostV1(w http.ResponseWriter, r *http.Request) {}
func createPostV1(w http.ResponseWriter, r *http.Request) {}
func addCommentV1(w http.ResponseWriter, r *http.Request) {}

// TODO: Document V2 handlers
func getPostsV2(w http.ResponseWriter, r *http.Request) {}
func getPostV2(w http.ResponseWriter, r *http.Request) {}
func createPostV2(w http.ResponseWriter, r *http.Request) {}
func uploadMedia(w http.ResponseWriter, r *http.Request) {}
func addCommentV2(w http.ResponseWriter, r *http.Request) {}
func getPostStats(w http.ResponseWriter, r *http.Request) {}

func main() {
    r := mux.NewRouter()

    // V1 routes
    v1 := r.PathPrefix("/api/v1").Subrouter()
    v1.HandleFunc("/posts", getPostsV1).Methods("GET")
    v1.HandleFunc("/posts/{id}", getPostV1).Methods("GET")
    v1.HandleFunc("/posts", createPostV1).Methods("POST")
    v1.HandleFunc("/posts/{id}/comments", addCommentV1).Methods("POST")

    // V2 routes
    v2 := r.PathPrefix("/api/v2").Subrouter()
    v2.HandleFunc("/posts", getPostsV2).Methods("GET")
    v2.HandleFunc("/posts/{id}", getPostV2).Methods("GET")
    v2.HandleFunc("/posts", createPostV2).Methods("POST")
    v2.HandleFunc("/posts/{id}/media", uploadMedia).Methods("POST")
    v2.HandleFunc("/posts/{id}/comments", addCommentV2).Methods("POST")
    v2.HandleFunc("/posts/{id}/stats", getPostStats).Methods("GET")

    // Swagger UI
    r.PathPrefix("/swagger/").Handler(httpSwagger.WrapHandler)

    http.ListenAndServe(":8080", r)
}
```

---

## Your Tasks

### Task 1: Document Both API Versions

Create two sets of general API info (you'll need two separate main files or use build tags):

**V1 API Info**:
- Title: "Blog API V1"
- Version: "1.0"
- Description: "Simple blog API with basic features"
- BasePath: "/api/v1"

**V2 API Info**:
- Title: "Blog API V2"
- Version: "2.0"
- Description: "Enhanced blog API with media, tags, and statistics"
- BasePath: "/api/v2"

### Task 2: Document All Models

Add comprehensive `example` tags:
- Simple fields: `example:"value"`
- Enums: `enums:"draft,published,archived"`
- Arrays: `example:"tag1,tag2"`
- Nested objects: Ensure child structs have examples
- Pointer fields: Handle nullable fields

### Task 3: Document Query Parameters

For GET /posts endpoints, document:
- All filter parameters
- Enum values where applicable
- Default values
- Min/max constraints (limit: max 100)

### Task 4: Document File Upload

For POST /posts/{id}/media:
- Use `formData` parameter type
- Document file parameter
- Show multipart/form-data
- Include file size limits in description

### Task 5: Document Nested Objects

Show proper relationships:
- PostV2 contains Author, Media, Comments
- CommentV2 contains Author
- Use `{object}` for nested types

### Task 6: Add Pagination Response

Document paginated response wrapper:
```go
type PaginatedResponse struct {
    Data       []PostV1 `json:"data"`
    Page       int      `json:"page"`
    Limit      int      `json:"limit"`
    TotalItems int      `json:"total_items"`
    TotalPages int      `json:"total_pages"`
}
```

---

## Special Documentation Patterns

### Enum Documentation

```go
// @Param status query string false "Filter by status" Enums(draft, published, archived)
```

### Array Response

```go
// @Success 200 {array} PostV1
```

### Nested Object Response

```go
// @Success 200 {object} PostV2
```

### File Upload

```go
// @Accept multipart/form-data
// @Param file formData file true "Media file"
```

### Nullable Fields

```go
PublishedAt *time.Time `json:"published_at,omitempty"`
```

---

## Expected Features in Swagger UI

### For V1 Endpoints
- Simple, flat structure
- Basic filtering
- Clear, minimal responses

### For V2 Endpoints
- Rich nested objects
- Complex filtering
- Media upload capability
- Statistics endpoint
- Enhanced models with all fields

### Interactive Testing
- Try both versions
- See different response structures
- Test file uploads
- Use all query parameters
- Verify enum dropdowns

---

## Bonus Challenges

### Bonus 1: Response Headers
Document custom headers:
```go
// @Header 200 {string} X-Total-Count "Total number of items"
// @Header 200 {string} X-Page "Current page"
```

### Bonus 2: Multiple Success Responses
Different responses based on conditions:
```go
// @Success 200 {object} PostV2 "Post found"
// @Success 204 "No posts found"
```

### Bonus 3: Detailed Error Responses
Document specific error codes:
```go
type ValidationError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

type ErrorResponseDetailed struct {
    Error   string            `json:"error"`
    Details []ValidationError `json:"details"`
}
```

### Bonus 4: API Examples
Add request/response examples:
```go
// @Success 200 {object} PostV2 "example post" "Post retrieved successfully"
```

### Bonus 5: Deprecation Notices
Mark V1 as deprecated:
```go
// @Deprecated
// @Description This endpoint is deprecated. Use V2 instead.
```

---

## Testing Checklist

- [ ] Both versions appear in Swagger UI
- [ ] V1 shows simple models
- [ ] V2 shows complex nested models
- [ ] Enums show as dropdowns
- [ ] File upload UI appears for media endpoint
- [ ] Query parameters have proper types
- [ ] Nested objects expand/collapse
- [ ] Example values populate correctly
- [ ] Array responses show correctly
- [ ] Status enums limited to valid values
- [ ] Both versions testable separately

---

## What You're Learning

✅ **Multi-version documentation** in same codebase  
✅ **Nested object schemas** with relationships  
✅ **Enum documentation** with dropdowns  
✅ **File upload endpoints** with multipart/form-data  
✅ **Complex filtering** with multiple parameters  
✅ **Pagination patterns** in responses  
✅ **Different response structures** per version  
✅ **Advanced Swagger features** for production APIs  

This demonstrates enterprise-level API documentation!
