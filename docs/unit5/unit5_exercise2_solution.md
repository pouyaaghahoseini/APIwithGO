# Unit 5 - Exercise 2 Solution: Document Multi-Version Blog API

**Complete implementation with comprehensive Swagger documentation**

---

## Full Solution Code

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

// @title           Blog API
// @version         2.0
// @description     A multi-version blog API with posts, comments, and media support
// @termsOfService  http://swagger.io/terms/

// @contact.name   API Support
// @contact.url    http://www.example.com/support
// @contact.email  support@example.com

// @license.name  MIT
// @license.url   https://opensource.org/licenses/MIT

// @host      localhost:8080
// @BasePath  /api

// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization
// @description Type "Bearer" followed by a space and JWT token.

// PostStatus represents the publication status of a post
type PostStatus string

const (
    StatusDraft     PostStatus = "draft"
    StatusPublished PostStatus = "published"
    StatusArchived  PostStatus = "archived"
)

// =============================================================================
// V1 MODELS
// =============================================================================

// PostV1 represents a blog post in V1 format (simple)
type PostV1 struct {
    ID        int       `json:"id" example:"1"`
    Title     string    `json:"title" example:"My First Blog Post"`
    Content   string    `json:"content" example:"This is the content of my post..."`
    Author    string    `json:"author" example:"johndoe"`
    Status    string    `json:"status" example:"published" enums:"draft,published,archived"`
    CreatedAt time.Time `json:"created_at" example:"2024-01-15T10:30:00Z"`
}

// CommentV1 represents a comment in V1 format
type CommentV1 struct {
    ID        int       `json:"id" example:"1"`
    PostID    int       `json:"post_id" example:"1"`
    Author    string    `json:"author" example:"janedoe"`
    Content   string    `json:"content" example:"Great post!"`
    CreatedAt time.Time `json:"created_at" example:"2024-01-15T11:00:00Z"`
}

// CreatePostV1Request is the request body for creating a post in V1
type CreatePostV1Request struct {
    Title   string `json:"title" example:"My Blog Post" binding:"required"`
    Content string `json:"content" example:"Post content here..." binding:"required"`
}

// =============================================================================
// V2 MODELS
// =============================================================================

// PostV2 represents a blog post in V2 format (enhanced)
type PostV2 struct {
    ID          int         `json:"id" example:"1"`
    Title       string      `json:"title" example:"My First Blog Post"`
    Content     string      `json:"content" example:"This is the full content..."`
    Excerpt     string      `json:"excerpt" example:"A short excerpt of the post"`
    Author      Author      `json:"author"`
    Status      PostStatus  `json:"status" example:"published" enums:"draft,published,archived"`
    Tags        []string    `json:"tags" example:"golang,api"`
    Media       []MediaFile `json:"media"`
    Comments    []CommentV2 `json:"comments"`
    Stats       PostStats   `json:"stats"`
    CreatedAt   time.Time   `json:"created_at" example:"2024-01-15T10:30:00Z"`
    UpdatedAt   time.Time   `json:"updated_at" example:"2024-01-15T10:30:00Z"`
    PublishedAt *time.Time  `json:"published_at,omitempty" example:"2024-01-15T10:30:00Z"`
}

// Author represents a blog post author
type Author struct {
    ID          int    `json:"id" example:"1"`
    Username    string `json:"username" example:"johndoe"`
    DisplayName string `json:"display_name" example:"John Doe"`
    Email       string `json:"email" example:"john@example.com"`
    AvatarURL   string `json:"avatar_url" example:"https://example.com/avatar.jpg"`
    Bio         string `json:"bio" example:"Software developer and blogger"`
}

// MediaFile represents an uploaded media file
type MediaFile struct {
    ID       int    `json:"id" example:"1"`
    Type     string `json:"type" example:"image" enums:"image,video,audio"`
    URL      string `json:"url" example:"https://example.com/media/photo.jpg"`
    Filename string `json:"filename" example:"photo.jpg"`
    Size     int64  `json:"size" example:"1024000"`
}

// CommentV2 represents a comment in V2 format
type CommentV2 struct {
    ID        int       `json:"id" example:"1"`
    PostID    int       `json:"post_id" example:"1"`
    Author    Author    `json:"author"`
    Content   string    `json:"content" example:"Great post!"`
    Likes     int       `json:"likes" example:"5"`
    CreatedAt time.Time `json:"created_at" example:"2024-01-15T11:00:00Z"`
}

// PostStats represents post engagement statistics
type PostStats struct {
    Views    int `json:"views" example:"1234"`
    Likes    int `json:"likes" example:"56"`
    Shares   int `json:"shares" example:"12"`
    Comments int `json:"comments" example:"8"`
}

// CreatePostV2Request is the request body for creating a post in V2
type CreatePostV2Request struct {
    Title   string     `json:"title" example:"My Blog Post" binding:"required"`
    Content string     `json:"content" example:"Full post content..." binding:"required"`
    Excerpt string     `json:"excerpt" example:"Short excerpt"`
    Tags    []string   `json:"tags" example:"golang,api"`
    Status  PostStatus `json:"status" example:"draft" enums:"draft,published,archived"`
}

// CreateCommentRequest is the request body for creating a comment
type CreateCommentRequest struct {
    Content string `json:"content" example:"Great article!" binding:"required"`
}

// PaginatedPostsV1 represents paginated V1 posts
type PaginatedPostsV1 struct {
    Data       []PostV1 `json:"data"`
    Page       int      `json:"page" example:"1"`
    Limit      int      `json:"limit" example:"10"`
    TotalItems int      `json:"total_items" example:"100"`
    TotalPages int      `json:"total_pages" example:"10"`
}

// PaginatedPostsV2 represents paginated V2 posts
type PaginatedPostsV2 struct {
    Data       []PostV2 `json:"data"`
    Page       int      `json:"page" example:"1"`
    Limit      int      `json:"limit" example:"10"`
    TotalItems int      `json:"total_items" example:"100"`
    TotalPages int      `json:"total_pages" example:"10"`
}

// ErrorResponse represents an error response
type ErrorResponse struct {
    Error string `json:"error" example:"Post not found"`
}

// Storage
var (
    postsV1     = make(map[int]PostV1)
    postsV2     = make(map[int]PostV2)
    commentsV1  = make(map[int][]CommentV1)
    commentsV2  = make(map[int][]CommentV2)
    authors     = make(map[int]Author)
    nextPostID  = 1
    nextCommID  = 1
    storageMu   sync.RWMutex
)

// =============================================================================
// V1 HANDLERS
// =============================================================================

// @Summary      List posts (V1)
// @Description  Get a paginated list of blog posts in V1 format
// @Tags         v1-posts
// @Accept       json
// @Produce      json
// @Param        status  query     string  false  "Filter by status"         Enums(draft, published, archived)
// @Param        author  query     string  false  "Filter by author username"
// @Param        search  query     string  false  "Search in title and content"
// @Param        sort    query     string  false  "Sort by field"            Enums(created_at, title)          default(created_at)
// @Param        order   query     string  false  "Sort order"               Enums(asc, desc)                  default(desc)
// @Param        page    query     int     false  "Page number"              default(1)
// @Param        limit   query     int     false  "Items per page (max 100)" default(10)
// @Success      200     {object}  PaginatedPostsV1
// @Failure      400     {object}  ErrorResponse
// @Failure      500     {object}  ErrorResponse
// @Router       /v1/posts [get]
func getPostsV1(w http.ResponseWriter, r *http.Request) {
    storageMu.RLock()
    defer storageMu.RUnlock()

    posts := []PostV1{}
    for _, post := range postsV1 {
        posts = append(posts, post)
    }

    // In real implementation: apply filters, sorting, pagination
    response := PaginatedPostsV1{
        Data:       posts,
        Page:       1,
        Limit:      10,
        TotalItems: len(posts),
        TotalPages: (len(posts) + 9) / 10,
    }

    respondJSON(w, http.StatusOK, response)
}

// @Summary      Get post by ID (V1)
// @Description  Get a single blog post in V1 format
// @Tags         v1-posts
// @Accept       json
// @Produce      json
// @Param        id   path      int  true  "Post ID"
// @Success      200  {object}  PostV1
// @Failure      404  {object}  ErrorResponse
// @Router       /v1/posts/{id} [get]
func getPostV1(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    storageMu.RLock()
    post, exists := postsV1[id]
    storageMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    respondJSON(w, http.StatusOK, post)
}

// @Summary      Create post (V1)
// @Description  Create a new blog post in V1 format
// @Tags         v1-posts
// @Accept       json
// @Produce      json
// @Param        post  body      CreatePostV1Request  true  "Post data"
// @Success      201   {object}  PostV1
// @Failure      400   {object}  ErrorResponse
// @Security     BearerAuth
// @Router       /v1/posts [post]
func createPostV1(w http.ResponseWriter, r *http.Request) {
    var req CreatePostV1Request
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    storageMu.Lock()
    post := PostV1{
        ID:        nextPostID,
        Title:     req.Title,
        Content:   req.Content,
        Author:    "johndoe", // Would come from auth
        Status:    "draft",
        CreatedAt: time.Now(),
    }
    postsV1[nextPostID] = post
    nextPostID++
    storageMu.Unlock()

    respondJSON(w, http.StatusCreated, post)
}

// @Summary      Add comment (V1)
// @Description  Add a comment to a blog post
// @Tags         v1-comments
// @Accept       json
// @Produce      json
// @Param        id       path      int                   true  "Post ID"
// @Param        comment  body      CreateCommentRequest  true  "Comment data"
// @Success      201      {object}  CommentV1
// @Failure      400      {object}  ErrorResponse
// @Failure      404      {object}  ErrorResponse
// @Security     BearerAuth
// @Router       /v1/posts/{id}/comments [post]
func addCommentV1(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    postID, _ := strconv.Atoi(vars["id"])

    storageMu.RLock()
    _, exists := postsV1[postID]
    storageMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    var req CreateCommentRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    storageMu.Lock()
    comment := CommentV1{
        ID:        nextCommID,
        PostID:    postID,
        Author:    "janedoe", // Would come from auth
        Content:   req.Content,
        CreatedAt: time.Now(),
    }
    commentsV1[postID] = append(commentsV1[postID], comment)
    nextCommID++
    storageMu.Unlock()

    respondJSON(w, http.StatusCreated, comment)
}

// =============================================================================
// V2 HANDLERS
// =============================================================================

// @Summary      List posts (V2)
// @Description  Get a paginated list of blog posts in V2 format with enhanced features
// @Tags         v2-posts
// @Accept       json
// @Produce      json
// @Param        status  query     string  false  "Filter by status"         Enums(draft, published, archived)
// @Param        author  query     string  false  "Filter by author username"
// @Param        tag     query     string  false  "Filter by tag"
// @Param        search  query     string  false  "Search in title and content"
// @Param        sort    query     string  false  "Sort by field"            Enums(created_at, updated_at, title, views)  default(created_at)
// @Param        order   query     string  false  "Sort order"               Enums(asc, desc)                             default(desc)
// @Param        page    query     int     false  "Page number"              default(1)
// @Param        limit   query     int     false  "Items per page (max 100)" default(10)
// @Success      200     {object}  PaginatedPostsV2
// @Failure      400     {object}  ErrorResponse
// @Failure      500     {object}  ErrorResponse
// @Header       200     {int}     X-Total-Count  "Total number of items"
// @Router       /v2/posts [get]
func getPostsV2(w http.ResponseWriter, r *http.Request) {
    storageMu.RLock()
    defer storageMu.RUnlock()

    posts := []PostV2{}
    for _, post := range postsV2 {
        posts = append(posts, post)
    }

    // Add custom header
    w.Header().Set("X-Total-Count", fmt.Sprintf("%d", len(posts)))

    response := PaginatedPostsV2{
        Data:       posts,
        Page:       1,
        Limit:      10,
        TotalItems: len(posts),
        TotalPages: (len(posts) + 9) / 10,
    }

    respondJSON(w, http.StatusOK, response)
}

// @Summary      Get post by ID (V2)
// @Description  Get a single blog post in V2 format with full details
// @Tags         v2-posts
// @Accept       json
// @Produce      json
// @Param        id   path      int  true  "Post ID"
// @Success      200  {object}  PostV2
// @Failure      404  {object}  ErrorResponse
// @Router       /v2/posts/{id} [get]
func getPostV2(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    storageMu.RLock()
    post, exists := postsV2[id]
    storageMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    respondJSON(w, http.StatusOK, post)
}

// @Summary      Create post (V2)
// @Description  Create a new blog post in V2 format with tags and status
// @Tags         v2-posts
// @Accept       json
// @Produce      json
// @Param        post  body      CreatePostV2Request  true  "Post data"
// @Success      201   {object}  PostV2
// @Failure      400   {object}  ErrorResponse
// @Security     BearerAuth
// @Router       /v2/posts [post]
func createPostV2(w http.ResponseWriter, r *http.Request) {
    var req CreatePostV2Request
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    now := time.Now()
    var publishedAt *time.Time
    if req.Status == StatusPublished {
        publishedAt = &now
    }

    storageMu.Lock()
    post := PostV2{
        ID:      nextPostID,
        Title:   req.Title,
        Content: req.Content,
        Excerpt: req.Excerpt,
        Author: Author{
            ID:          1,
            Username:    "johndoe",
            DisplayName: "John Doe",
            Email:       "john@example.com",
        },
        Status:      req.Status,
        Tags:        req.Tags,
        Media:       []MediaFile{},
        Comments:    []CommentV2{},
        Stats:       PostStats{},
        CreatedAt:   now,
        UpdatedAt:   now,
        PublishedAt: publishedAt,
    }
    postsV2[nextPostID] = post
    nextPostID++
    storageMu.Unlock()

    respondJSON(w, http.StatusCreated, post)
}

// @Summary      Upload media
// @Description  Upload a media file (image, video, audio) to a post
// @Tags         v2-media
// @Accept       multipart/form-data
// @Produce      json
// @Param        id    path      int   true  "Post ID"
// @Param        file  formData  file  true  "Media file (max 10MB)"
// @Success      201   {object}  MediaFile
// @Failure      400   {object}  ErrorResponse
// @Failure      404   {object}  ErrorResponse
// @Security     BearerAuth
// @Router       /v2/posts/{id}/media [post]
func uploadMedia(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    postID, _ := strconv.Atoi(vars["id"])

    storageMu.RLock()
    _, exists := postsV2[postID]
    storageMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    // Parse multipart form (max 10MB)
    err := r.ParseMultipartForm(10 << 20)
    if err != nil {
        respondError(w, http.StatusBadRequest, "File too large")
        return
    }

    file, header, err := r.FormFile("file")
    if err != nil {
        respondError(w, http.StatusBadRequest, "File is required")
        return
    }
    defer file.Close()

    // In real implementation: save file, detect type, generate URL
    media := MediaFile{
        ID:       1,
        Type:     "image",
        URL:      "https://example.com/media/uploaded.jpg",
        Filename: header.Filename,
        Size:     header.Size,
    }

    respondJSON(w, http.StatusCreated, media)
}

// @Summary      Add comment (V2)
// @Description  Add a comment to a blog post with author details
// @Tags         v2-comments
// @Accept       json
// @Produce      json
// @Param        id       path      int                   true  "Post ID"
// @Param        comment  body      CreateCommentRequest  true  "Comment data"
// @Success      201      {object}  CommentV2
// @Failure      400      {object}  ErrorResponse
// @Failure      404      {object}  ErrorResponse
// @Security     BearerAuth
// @Router       /v2/posts/{id}/comments [post]
func addCommentV2(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    postID, _ := strconv.Atoi(vars["id"])

    storageMu.RLock()
    _, exists := postsV2[postID]
    storageMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    var req CreateCommentRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    storageMu.Lock()
    comment := CommentV2{
        ID:     nextCommID,
        PostID: postID,
        Author: Author{
            ID:          2,
            Username:    "janedoe",
            DisplayName: "Jane Doe",
            Email:       "jane@example.com",
        },
        Content:   req.Content,
        Likes:     0,
        CreatedAt: time.Now(),
    }
    commentsV2[postID] = append(commentsV2[postID], comment)
    nextCommID++
    storageMu.Unlock()

    respondJSON(w, http.StatusCreated, comment)
}

// @Summary      Get post statistics
// @Description  Get engagement statistics for a blog post
// @Tags         v2-statistics
// @Accept       json
// @Produce      json
// @Param        id   path      int  true  "Post ID"
// @Success      200  {object}  PostStats
// @Failure      404  {object}  ErrorResponse
// @Router       /v2/posts/{id}/stats [get]
func getPostStats(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    storageMu.RLock()
    post, exists := postsV2[id]
    storageMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    respondJSON(w, http.StatusOK, post.Stats)
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
    respondJSON(w, status, ErrorResponse{Error: message})
}

func seedDatabase() {
    storageMu.Lock()
    defer storageMu.Unlock()

    now := time.Now()

    // Seed V1 posts
    postsV1[1] = PostV1{
        ID:        1,
        Title:     "Getting Started with Go",
        Content:   "Go is a great language for building APIs...",
        Author:    "johndoe",
        Status:    "published",
        CreatedAt: now,
    }

    // Seed V2 posts
    postsV2[1] = PostV2{
        ID:      1,
        Title:   "Getting Started with Go",
        Content: "Go is a great language for building APIs...",
        Excerpt: "Learn the basics of Go programming",
        Author: Author{
            ID:          1,
            Username:    "johndoe",
            DisplayName: "John Doe",
            Email:       "john@example.com",
            AvatarURL:   "https://example.com/avatars/john.jpg",
            Bio:         "Software developer",
        },
        Status:      StatusPublished,
        Tags:        []string{"golang", "tutorial"},
        Media:       []MediaFile{},
        Comments:    []CommentV2{},
        Stats:       PostStats{Views: 1234, Likes: 56, Shares: 12, Comments: 8},
        CreatedAt:   now,
        UpdatedAt:   now,
        PublishedAt: &now,
    }

    nextPostID = 2
}

func main() {
    seedDatabase()

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

    fmt.Println("Server starting on :8080")
    fmt.Println("API V1: http://localhost:8080/api/v1")
    fmt.Println("API V2: http://localhost:8080/api/v2")
    fmt.Println("Swagger UI: http://localhost:8080/swagger/index.html")
    http.ListenAndServe(":8080", r)
}
```

---

## Key Concepts Explained

### 1. Enum Documentation

```go
type PostStatus string

const (
    StatusDraft     PostStatus = "draft"
    StatusPublished PostStatus = "published"
    StatusArchived  PostStatus = "archived"
)

// In struct:
Status PostStatus `json:"status" example:"published" enums:"draft,published,archived"`

// In handler annotation:
// @Param status query string false "Filter by status" Enums(draft, published, archived)
```

**Creates**: Dropdown in Swagger UI with only valid values.

### 2. Nested Objects

```go
// PostV2 contains nested objects
type PostV2 struct {
    Author   Author      `json:"author"`      // Nested object
    Media    []MediaFile `json:"media"`       // Array of nested objects
    Comments []CommentV2 `json:"comments"`    // Array of nested objects
    Stats    PostStats   `json:"stats"`       // Nested object
}

// Handler annotation:
// @Success 200 {object} PostV2
```

**Swagger generates**: Full schema with expandable nested objects.

### 3. Nullable Fields

```go
PublishedAt *time.Time `json:"published_at,omitempty" example:"2024-01-15T10:30:00Z"`
```

**Pointer types**: Allow null values in JSON.
**omitempty**: Excludes field if null.

### 4. File Upload Documentation

```go
// @Accept multipart/form-data
// @Param file formData file true "Media file (max 10MB)"
```

**Creates**: File upload UI in Swagger with browse button.

### 5. Pagination Response

```go
type PaginatedPostsV2 struct {
    Data       []PostV2 `json:"data"`
    Page       int      `json:"page" example:"1"`
    Limit      int      `json:"limit" example:"10"`
    TotalItems int      `json:"total_items" example:"100"`
    TotalPages int      `json:"total_pages" example:"10"`
}

// @Success 200 {object} PaginatedPostsV2
```

**Shows**: Full pagination metadata structure.

### 6. Custom Response Headers

```go
// @Header 200 {int} X-Total-Count "Total number of items"
```

**Documents**: Custom headers in response.

### 7. Multiple Query Parameters

```go
// @Param status  query string false "Filter by status" Enums(draft, published, archived)
// @Param author  query string false "Filter by author username"
// @Param tag     query string false "Filter by tag"
// @Param search  query string false "Search in title and content"
// @Param sort    query string false "Sort by field" Enums(created_at, updated_at, title, views) default(created_at)
// @Param order   query string false "Sort order" Enums(asc, desc) default(desc)
// @Param page    query int    false "Page number" default(1)
// @Param limit   query int    false "Items per page (max 100)" default(10)
```

**Creates**: Form with all filters, dropdowns for enums, defaults pre-filled.

### 8. Tag Organization

```go
// @Tags v1-posts
// @Tags v2-posts
// @Tags v2-comments
// @Tags v2-media
// @Tags v2-statistics
```

**Groups**: Endpoints by version and resource type.

---

## Swagger UI Features

### V1 Section
- **v1-posts**: Simple CRUD operations
- **v1-comments**: Basic comment system
- Simple, flat response structures

### V2 Section
- **v2-posts**: Enhanced with nested objects
- **v2-comments**: Full author details
- **v2-media**: File upload capability
- **v2-statistics**: Engagement metrics
- Rich, nested response structures

### Interactive Features
1. **Expand/Collapse**: All endpoint groups
2. **Try it out**: Test any endpoint
3. **Authorize**: Add Bearer token
4. **Upload files**: Browse and select
5. **Enum dropdowns**: Select from valid values
6. **Default values**: Pre-filled parameters
7. **Example responses**: See expected output

---

## Response Structure Comparison

### V1 Response (Simple)
```json
{
  "id": 1,
  "title": "Getting Started with Go",
  "content": "Go is a great language...",
  "author": "johndoe",
  "status": "published",
  "created_at": "2024-01-15T10:30:00Z"
}
```

### V2 Response (Enhanced)
```json
{
  "id": 1,
  "title": "Getting Started with Go",
  "content": "Go is a great language...",
  "excerpt": "Learn the basics of Go",
  "author": {
    "id": 1,
    "username": "johndoe",
    "display_name": "John Doe",
    "email": "john@example.com",
    "avatar_url": "https://example.com/avatars/john.jpg",
    "bio": "Software developer"
  },
  "status": "published",
  "tags": ["golang", "tutorial"],
  "media": [],
  "comments": [],
  "stats": {
    "views": 1234,
    "likes": 56,
    "shares": 12,
    "comments": 8
  },
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z",
  "published_at": "2024-01-15T10:30:00Z"
}
```

---

## Testing Guide

### 1. View Documentation
Visit: `http://localhost:8080/swagger/index.html`

### 2. Explore Both Versions
- Expand "v1-posts" section
- Expand "v2-posts" section
- Compare response structures

### 3. Test V1 Endpoints
- GET /v1/posts - See pagination
- GET /v1/posts/{id} - Simple response
- Try filters (status, author)

### 4. Test V2 Endpoints
- GET /v2/posts - Enhanced response
- Try all filters (status, author, tag, search)
- GET /v2/posts/{id} - Full nested response

### 5. Test Authentication
- Click "Authorize"
- Enter: `Bearer fake-token-for-testing`
- Try POST endpoints

### 6. Test File Upload
- POST /v2/posts/{id}/media
- Click "Try it out"
- Browse and select file
- Execute

### 7. Test Enums
- Open any endpoint with enums
- See dropdown with valid values only
- Select from dropdown

---

## What You've Learned

✅ **Multi-version documentation** in single API  
✅ **Nested object schemas** with proper relationships  
✅ **Enum types** with dropdown selection  
✅ **File upload** endpoints with multipart/form-data  
✅ **Pagination** response structures  
✅ **Custom headers** in responses  
✅ **Complex filtering** with multiple parameters  
✅ **Nullable fields** with pointer types  
✅ **Tag organization** for clear grouping  
✅ **Response structure evolution** across versions  

You now know how to document enterprise-level APIs with complex structures!
