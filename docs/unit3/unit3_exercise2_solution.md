# Unit 3 - Exercise 2 Solution: Multi-Tenant Blog Platform with RBAC

**Complete implementation with explanations**

---

## Full Solution Code

Due to the complexity, this solution is organized into logical sections.

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "strconv"
    "strings"
    "sync"
    "time"

    "github.com/golang-jwt/jwt/v5"
    "github.com/gorilla/mux"
    "golang.org/x/crypto/bcrypt"
)

var jwtSecret = []byte("your-secret-key")

// =============================================================================
// MODELS
// =============================================================================

type User struct {
    ID           int       `json:"id"`
    Username     string    `json:"username"`
    Email        string    `json:"email"`
    PasswordHash string    `json:"-"`
    Role         string    `json:"role"`
    CreatedAt    time.Time `json:"created_at"`
}

type Post struct {
    ID        int       `json:"id"`
    Title     string    `json:"title"`
    Content   string    `json:"content"`
    AuthorID  int       `json:"author_id"`
    Author    User      `json:"author"`
    Published bool      `json:"published"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

type Comment struct {
    ID        int       `json:"id"`
    PostID    int       `json:"post_id"`
    UserID    int       `json:"user_id"`
    User      User      `json:"user"`
    Content   string    `json:"content"`
    Approved  bool      `json:"approved"`
    CreatedAt time.Time `json:"created_at"`
}

type Claims struct {
    UserID   int    `json:"user_id"`
    Username string `json:"username"`
    Role     string `json:"role"`
    jwt.RegisteredClaims
}

// =============================================================================
// STORAGE
// =============================================================================

var (
    users         = make(map[int]User)
    posts         = make(map[int]Post)
    comments      = make(map[int]Comment)
    usernames     = make(map[string]int)
    nextUserID    = 1
    nextPostID    = 1
    nextCommentID = 1
    usersMu       sync.RWMutex
    postsMu       sync.RWMutex
    commentsMu    sync.RWMutex
)

type contextKey string

const UserContextKey contextKey = "user"

// =============================================================================
// AUTHENTICATION HANDLERS
// =============================================================================

func register(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Username string `json:"username"`
        Email    string `json:"email"`
        Password string `json:"password"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Check uniqueness
    usersMu.RLock()
    _, exists := usernames[req.Username]
    usersMu.RUnlock()

    if exists {
        respondError(w, http.StatusConflict, "Username already exists")
        return
    }

    // Hash password
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
    if err != nil {
        respondError(w, http.StatusInternalServerError, "Failed to process password")
        return
    }

    // Create user
    usersMu.Lock()
    user := User{
        ID:           nextUserID,
        Username:     req.Username,
        Email:        req.Email,
        PasswordHash: string(hashedPassword),
        Role:         "user", // Default role
        CreatedAt:    time.Now(),
    }
    users[nextUserID] = user
    usernames[req.Username] = nextUserID
    nextUserID++
    usersMu.Unlock()

    // Generate token
    token, _ := generateToken(user.ID, user.Username, user.Role)

    respondJSON(w, http.StatusCreated, map[string]interface{}{
        "user":  user,
        "token": token,
    })
}

func login(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Username string `json:"username"`
        Password string `json:"password"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Find user
    usersMu.RLock()
    userID, exists := usernames[req.Username]
    if !exists {
        usersMu.RUnlock()
        respondError(w, http.StatusUnauthorized, "Invalid credentials")
        return
    }
    user := users[userID]
    usersMu.RUnlock()

    // Verify password
    err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(req.Password))
    if err != nil {
        respondError(w, http.StatusUnauthorized, "Invalid credentials")
        return
    }

    // Generate token
    token, _ := generateToken(user.ID, user.Username, user.Role)

    respondJSON(w, http.StatusOK, map[string]interface{}{
        "token": token,
        "user":  user,
    })
}

// =============================================================================
// POST HANDLERS
// =============================================================================

func getPosts(w http.ResponseWriter, r *http.Request) {
    user, authenticated := getUserFromContext(r)

    postsMu.RLock()
    usersMu.RLock()
    defer postsMu.RUnlock()
    defer usersMu.RUnlock()

    var results []Post
    for _, post := range posts {
        // Admin and moderator see everything
        if authenticated && (user.Role == "admin" || user.Role == "moderator") {
            post.Author = users[post.AuthorID]
            results = append(results, post)
            continue
        }

        // Published posts visible to all
        if post.Published {
            post.Author = users[post.AuthorID]
            results = append(results, post)
            continue
        }

        // Unpublished posts only visible to author
        if authenticated && user.UserID == post.AuthorID {
            post.Author = users[post.AuthorID]
            results = append(results, post)
        }
    }

    respondJSON(w, http.StatusOK, results)
}

func getPost(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    postsMu.RLock()
    post, exists := posts[id]
    postsMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    user, authenticated := getUserFromContext(r)

    // Check if user can view this post
    canView := post.Published ||
        (authenticated && user.UserID == post.AuthorID) ||
        (authenticated && (user.Role == "admin" || user.Role == "moderator"))

    if !canView {
        respondError(w, http.StatusForbidden, "Access denied")
        return
    }

    usersMu.RLock()
    post.Author = users[post.AuthorID]
    usersMu.RUnlock()

    respondJSON(w, http.StatusOK, post)
}

func createPost(w http.ResponseWriter, r *http.Request) {
    user, ok := getUserFromContext(r)
    if !ok {
        respondError(w, http.StatusUnauthorized, "Unauthorized")
        return
    }

    var req struct {
        Title   string `json:"title"`
        Content string `json:"content"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Validate
    if len(req.Title) < 5 {
        respondError(w, http.StatusBadRequest, "Title must be at least 5 characters")
        return
    }

    if len(req.Content) < 10 {
        respondError(w, http.StatusBadRequest, "Content must be at least 10 characters")
        return
    }

    // Create post
    now := time.Now()
    postsMu.Lock()
    post := Post{
        ID:        nextPostID,
        Title:     req.Title,
        Content:   req.Content,
        AuthorID:  user.UserID,
        Published: false,
        CreatedAt: now,
        UpdatedAt: now,
    }
    posts[nextPostID] = post
    nextPostID++
    postsMu.Unlock()

    usersMu.RLock()
    post.Author = users[user.UserID]
    usersMu.RUnlock()

    respondJSON(w, http.StatusCreated, post)
}

func updatePost(w http.ResponseWriter, r *http.Request) {
    user, ok := getUserFromContext(r)
    if !ok {
        respondError(w, http.StatusUnauthorized, "Unauthorized")
        return
    }

    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    postsMu.RLock()
    post, exists := posts[id]
    postsMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    // Check permission
    if !canEdit(user, post) {
        respondError(w, http.StatusForbidden, "You don't have permission to edit this post")
        return
    }

    var req struct {
        Title   string `json:"title"`
        Content string `json:"content"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Update post
    postsMu.Lock()
    post.Title = req.Title
    post.Content = req.Content
    post.UpdatedAt = time.Now()
    posts[id] = post
    postsMu.Unlock()

    usersMu.RLock()
    post.Author = users[post.AuthorID]
    usersMu.RUnlock()

    respondJSON(w, http.StatusOK, post)
}

func deletePost(w http.ResponseWriter, r *http.Request) {
    user, ok := getUserFromContext(r)
    if !ok {
        respondError(w, http.StatusUnauthorized, "Unauthorized")
        return
    }

    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    postsMu.RLock()
    post, exists := posts[id]
    postsMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    // Check permission
    if !canDelete(user, post) {
        respondError(w, http.StatusForbidden, "You don't have permission to delete this post")
        return
    }

    postsMu.Lock()
    delete(posts, id)
    postsMu.Unlock()

    w.WriteHeader(http.StatusNoContent)
}

func publishPost(w http.ResponseWriter, r *http.Request) {
    user, ok := getUserFromContext(r)
    if !ok {
        respondError(w, http.StatusUnauthorized, "Unauthorized")
        return
    }

    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    postsMu.RLock()
    post, exists := posts[id]
    postsMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    // Check permission
    if !canPublish(user, post) {
        respondError(w, http.StatusForbidden, "You don't have permission to publish this post")
        return
    }

    postsMu.Lock()
    post.Published = true
    post.UpdatedAt = time.Now()
    posts[id] = post
    postsMu.Unlock()

    usersMu.RLock()
    post.Author = users[post.AuthorID]
    usersMu.RUnlock()

    respondJSON(w, http.StatusOK, post)
}

func unpublishPost(w http.ResponseWriter, r *http.Request) {
    user, ok := getUserFromContext(r)
    if !ok {
        respondError(w, http.StatusUnauthorized, "Unauthorized")
        return
    }

    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    postsMu.RLock()
    post, exists := posts[id]
    postsMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    // Check permission (same as publish)
    if !canPublish(user, post) {
        respondError(w, http.StatusForbidden, "You don't have permission to unpublish this post")
        return
    }

    postsMu.Lock()
    post.Published = false
    post.UpdatedAt = time.Now()
    posts[id] = post
    postsMu.Unlock()

    usersMu.RLock()
    post.Author = users[post.AuthorID]
    usersMu.RUnlock()

    respondJSON(w, http.StatusOK, post)
}

// =============================================================================
// COMMENT HANDLERS
// =============================================================================

func getComments(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    postID, _ := strconv.Atoi(vars["id"])

    user, authenticated := getUserFromContext(r)

    commentsMu.RLock()
    usersMu.RLock()
    defer commentsMu.RUnlock()
    defer usersMu.RUnlock()

    var results []Comment
    for _, comment := range comments {
        if comment.PostID != postID {
            continue
        }

        // Show approved comments to everyone
        if comment.Approved {
            comment.User = users[comment.UserID]
            results = append(results, comment)
            continue
        }

        // Show unapproved comments to moderators/admins
        if authenticated && (user.Role == "admin" || user.Role == "moderator") {
            comment.User = users[comment.UserID]
            results = append(results, comment)
        }
    }

    respondJSON(w, http.StatusOK, results)
}

func createComment(w http.ResponseWriter, r *http.Request) {
    user, ok := getUserFromContext(r)
    if !ok {
        respondError(w, http.StatusUnauthorized, "Unauthorized")
        return
    }

    vars := mux.Vars(r)
    postID, _ := strconv.Atoi(vars["id"])

    // Check post exists
    postsMu.RLock()
    _, exists := posts[postID]
    postsMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    var req struct {
        Content string `json:"content"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Validate
    if len(req.Content) < 1 {
        respondError(w, http.StatusBadRequest, "Content is required")
        return
    }

    // Create comment (unapproved by default)
    commentsMu.Lock()
    comment := Comment{
        ID:        nextCommentID,
        PostID:    postID,
        UserID:    user.UserID,
        Content:   req.Content,
        Approved:  false,
        CreatedAt: time.Now(),
    }
    comments[nextCommentID] = comment
    nextCommentID++
    commentsMu.Unlock()

    usersMu.RLock()
    comment.User = users[user.UserID]
    usersMu.RUnlock()

    respondJSON(w, http.StatusCreated, comment)
}

func deleteComment(w http.ResponseWriter, r *http.Request) {
    user, ok := getUserFromContext(r)
    if !ok {
        respondError(w, http.StatusUnauthorized, "Unauthorized")
        return
    }

    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    commentsMu.RLock()
    comment, exists := comments[id]
    commentsMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Comment not found")
        return
    }

    // Check permission: own comment OR moderator/admin
    canDelete := comment.UserID == user.UserID ||
        user.Role == "admin" ||
        user.Role == "moderator"

    if !canDelete {
        respondError(w, http.StatusForbidden, "You don't have permission to delete this comment")
        return
    }

    commentsMu.Lock()
    delete(comments, id)
    commentsMu.Unlock()

    w.WriteHeader(http.StatusNoContent)
}

func approveComment(w http.ResponseWriter, r *http.Request) {
    user, ok := getUserFromContext(r)
    if !ok {
        respondError(w, http.StatusUnauthorized, "Unauthorized")
        return
    }

    // Check permission
    if !canApprove(user) {
        respondError(w, http.StatusForbidden, "You don't have permission to approve comments")
        return
    }

    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    commentsMu.Lock()
    comment, exists := comments[id]
    if !exists {
        commentsMu.Unlock()
        respondError(w, http.StatusNotFound, "Comment not found")
        return
    }

    comment.Approved = true
    comments[id] = comment
    commentsMu.Unlock()

    usersMu.RLock()
    comment.User = users[comment.UserID]
    usersMu.RUnlock()

    respondJSON(w, http.StatusOK, comment)
}

// =============================================================================
// ADMIN HANDLERS
// =============================================================================

func getAllUsers(w http.ResponseWriter, r *http.Request) {
    usersMu.RLock()
    defer usersMu.RUnlock()

    userList := make([]User, 0, len(users))
    for _, user := range users {
        userList = append(userList, user)
    }

    respondJSON(w, http.StatusOK, userList)
}

func changeUserRole(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    var req struct {
        Role string `json:"role"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Validate role
    validRoles := map[string]bool{
        "user": true, "moderator": true, "admin": true,
    }

    if !validRoles[req.Role] {
        respondError(w, http.StatusBadRequest, "Invalid role")
        return
    }

    usersMu.Lock()
    user, exists := users[id]
    if !exists {
        usersMu.Unlock()
        respondError(w, http.StatusNotFound, "User not found")
        return
    }

    user.Role = req.Role
    users[id] = user
    usersMu.Unlock()

    respondJSON(w, http.StatusOK, user)
}

func getStats(w http.ResponseWriter, r *http.Request) {
    usersMu.RLock()
    totalUsers := len(users)
    usersMu.RUnlock()

    postsMu.RLock()
    totalPosts := len(posts)
    publishedPosts := 0
    for _, post := range posts {
        if post.Published {
            publishedPosts++
        }
    }
    postsMu.RUnlock()

    commentsMu.RLock()
    totalComments := len(comments)
    approvedComments := 0
    for _, comment := range comments {
        if comment.Approved {
            approvedComments++
        }
    }
    commentsMu.RUnlock()

    stats := map[string]interface{}{
        "total_users":       totalUsers,
        "total_posts":       totalPosts,
        "published_posts":   publishedPosts,
        "total_comments":    totalComments,
        "approved_comments": approvedComments,
    }

    respondJSON(w, http.StatusOK, stats)
}

// =============================================================================
// PERMISSION HELPERS
// =============================================================================

func canEdit(user *Claims, post Post) bool {
    if user == nil {
        return false
    }

    // Admins and moderators can edit anything
    if user.Role == "admin" || user.Role == "moderator" {
        return true
    }

    // Users can edit their own posts
    return user.UserID == post.AuthorID
}

func canDelete(user *Claims, post Post) bool {
    if user == nil {
        return false
    }

    // Admins and moderators can delete anything
    if user.Role == "admin" || user.Role == "moderator" {
        return true
    }

    // Users can delete their own posts
    return user.UserID == post.AuthorID
}

func canPublish(user *Claims, post Post) bool {
    if user == nil {
        return false
    }

    // Admins can publish anything
    if user.Role == "admin" {
        return true
    }

    // Moderators CANNOT publish (key difference!)
    if user.Role == "moderator" {
        return false
    }

    // Users can publish their own posts
    return user.UserID == post.AuthorID
}

func canApprove(user *Claims) bool {
    if user == nil {
        return false
    }

    // Only admins and moderators can approve comments
    return user.Role == "admin" || user.Role == "moderator"
}

// =============================================================================
// JWT FUNCTIONS
// =============================================================================

func generateToken(userID int, username, role string) (string, error) {
    claims := Claims{
        UserID:   userID,
        Username: username,
        Role:     role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}

func validateToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        return jwtSecret, nil
    })

    if err != nil {
        return nil, err
    }

    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }

    return nil, fmt.Errorf("invalid token")
}

// =============================================================================
// MIDDLEWARE
// =============================================================================

func optionalAuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            next.ServeHTTP(w, r)
            return
        }

        parts := strings.Split(authHeader, " ")
        if len(parts) == 2 && parts[0] == "Bearer" {
            claims, err := validateToken(parts[1])
            if err == nil {
                ctx := context.WithValue(r.Context(), UserContextKey, claims)
                next.ServeHTTP(w, r.WithContext(ctx))
                return
            }
        }

        next.ServeHTTP(w, r)
    })
}

func requireAuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            respondError(w, http.StatusUnauthorized, "Authorization required")
            return
        }

        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            respondError(w, http.StatusUnauthorized, "Invalid authorization header")
            return
        }

        claims, err := validateToken(parts[1])
        if err != nil {
            respondError(w, http.StatusUnauthorized, "Invalid or expired token")
            return
        }

        ctx := context.WithValue(r.Context(), UserContextKey, claims)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func requireRole(roles ...string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            user, ok := getUserFromContext(r)
            if !ok {
                respondError(w, http.StatusUnauthorized, "Unauthorized")
                return
            }

            hasRole := false
            for _, role := range roles {
                if user.Role == role {
                    hasRole = true
                    break
                }
            }

            if !hasRole {
                respondError(w, http.StatusForbidden, "Insufficient permissions")
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}

// =============================================================================
// HELPERS
// =============================================================================

func getUserFromContext(r *http.Request) (*Claims, bool) {
    user, ok := r.Context().Value(UserContextKey).(*Claims)
    return user, ok
}

func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}

// =============================================================================
// MAIN
// =============================================================================

func main() {
    r := mux.NewRouter()

    // Public routes
    r.HandleFunc("/register", register).Methods("POST")
    r.HandleFunc("/login", login).Methods("POST")

    // Posts with optional auth
    r.HandleFunc("/posts", getPosts).Methods("GET")
    r.HandleFunc("/posts/{id}", getPost).Methods("GET")

    // Protected post routes
    protectedPosts := r.PathPrefix("/posts").Subrouter()
    protectedPosts.Use(requireAuthMiddleware)
    protectedPosts.HandleFunc("", createPost).Methods("POST")
    protectedPosts.HandleFunc("/{id}", updatePost).Methods("PUT")
    protectedPosts.HandleFunc("/{id}", deletePost).Methods("DELETE")
    protectedPosts.HandleFunc("/{id}/publish", publishPost).Methods("POST")
    protectedPosts.HandleFunc("/{id}/unpublish", unpublishPost).Methods("POST")

    // Comments
    r.HandleFunc("/posts/{id}/comments", getComments).Methods("GET")

    protectedComments := r.NewRoute().Subrouter()
    protectedComments.Use(requireAuthMiddleware)
    protectedComments.HandleFunc("/posts/{id}/comments", createComment).Methods("POST")
    protectedComments.HandleFunc("/comments/{id}", deleteComment).Methods("DELETE")

    // Moderator/Admin only
    modRoutes := r.PathPrefix("/comments").Subrouter()
    modRoutes.Use(requireAuthMiddleware)
    modRoutes.Use(requireRole("moderator", "admin"))
    modRoutes.HandleFunc("/{id}/approve", approveComment).Methods("POST")

    // Admin routes
    adminRoutes := r.PathPrefix("/admin").Subrouter()
    adminRoutes.Use(requireAuthMiddleware)
    adminRoutes.Use(requireRole("admin"))
    adminRoutes.HandleFunc("/users", getAllUsers).Methods("GET")
    adminRoutes.HandleFunc("/users/{id}/role", changeUserRole).Methods("PUT")
    adminRoutes.HandleFunc("/stats", getStats).Methods("GET")

    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", r)
}
```

---

## Key RBAC Concepts Explained

### 1. Permission Helper Functions

The core of RBAC - these functions encapsulate authorization logic:

```go
func canPublish(user *Claims, post Post) bool {
    if user == nil {
        return false
    }

    // Admin: can publish anything
    if user.Role == "admin" {
        return true
    }

    // Moderator: CANNOT publish (key business rule)
    if user.Role == "moderator" {
        return false
    }

    // User: can only publish own posts
    return user.UserID == post.AuthorID
}
```

**Why this pattern?**
- Centralizes authorization logic
- Makes permissions easy to audit
- Changes in one place affect whole system

### 2. Optional vs Required Authentication

```go
// Optional - allows both authenticated and guest access
func optionalAuthMiddleware(next http.Handler) http.Handler {
    // Try to extract token, but don't fail if missing
    if authHeader == "" {
        next.ServeHTTP(w, r)  // Continue without user
        return
    }
    // ...
}

// Required - blocks unauthenticated users
func requireAuthMiddleware(next http.Handler) http.Handler {
    if authHeader == "" {
        respondError(w, 401, "Authorization required")
        return
    }
    // ...
}
```

**Use cases**:
- Optional: Public content that guests can view
- Required: Actions that need identity

### 3. Resource Ownership Checks

```go
func updatePost(w http.ResponseWriter, r *http.Request) {
    user, _ := getUserFromContext(r)
    
    // Get the post
    post := posts[id]
    
    // Check if user can edit THIS specific post
    if !canEdit(user, post) {
        respondError(w, 403, "Access denied")
        return
    }
    
    // Proceed with update
}
```

**Pattern**: Always check permissions against the specific resource, not just user role.

### 4. Filtering by Permission

```go
func getPosts(w http.ResponseWriter, r *http.Request) {
    user, authenticated := getUserFromContext(r)
    
    var results []Post
    for _, post := range posts {
        // Different visibility rules per role
        if user.Role == "admin" || user.Role == "moderator" {
            results = append(results, post)  // See everything
        } else if post.Published {
            results = append(results, post)  // Public
        } else if authenticated && user.UserID == post.AuthorID {
            results = append(results, post)  // Own unpublished
        }
    }
    
    return results
}
```

**Key**: Data filtering based on who's asking, not just what they're asking for.

### 5. Multi-Role Middleware

```go
func requireRole(roles ...string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            user, _ := getUserFromContext(r)
            
            // Check if user has ANY of the allowed roles
            hasRole := false
            for _, role := range roles {
                if user.Role == role {
                    hasRole = true
                    break
                }
            }
            
            if !hasRole {
                respondError(w, 403, "Insufficient permissions")
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}

// Usage
modRoutes.Use(requireRole("moderator", "admin"))
```

---

## Permission Matrix Implementation

| Action | Implementation |
|--------|----------------|
| View published posts | `if post.Published` - no auth required |
| View own unpublished | `if authenticated && user.ID == post.AuthorID` |
| View any unpublished | `if user.Role == "admin" \|\| user.Role == "moderator"` |
| Edit post | `canEdit()` - owner OR mod/admin |
| Delete post | `canDelete()` - owner OR mod/admin |
| Publish post | `canPublish()` - admin always, user if owner, **mod never** |
| Approve comment | `canApprove()` - mod/admin only |

---

## Testing Complete RBAC Flow

```bash
# Setup: Create users with different roles
curl -X POST http://localhost:8080/register \
  -d '{"username":"alice","email":"alice@test.com","password":"Pass123"}'
# Save as USER_TOKEN

curl -X POST http://localhost:8080/register \
  -d '{"username":"bob","email":"bob@test.com","password":"Pass123"}'  
# Promote bob to moderator using admin

curl -X POST http://localhost:8080/register \
  -d '{"username":"admin","email":"admin@test.com","password":"Pass123"}'
# Promote to admin, save as ADMIN_TOKEN

# Promote roles
curl -X PUT http://localhost:8080/admin/users/2/role \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"role":"moderator"}'

curl -X PUT http://localhost:8080/admin/users/3/role \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"role":"admin"}'

# Test permissions
# Alice creates post
curl -X POST http://localhost:8080/posts \
  -H "Authorization: Bearer $USER_TOKEN" \
  -d '{"title":"Alice Post","content":"Content here"}'

# Alice publishes (should work - own post)
curl -X POST http://localhost:8080/posts/1/publish \
  -H "Authorization: Bearer $USER_TOKEN"

# Bob (mod) creates post  
curl -X POST http://localhost:8080/posts \
  -H "Authorization: Bearer $MOD_TOKEN" \
  -d '{"title":"Bob Post","content":"Content here"}'

# Bob tries to publish own post (should FAIL - mods can't publish)
curl -X POST http://localhost:8080/posts/2/publish \
  -H "Authorization: Bearer $MOD_TOKEN"
# Expected: 403 Forbidden

# Admin publishes Bob's post (should work)
curl -X POST http://localhost:8080/posts/2/publish \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Bob edits Alice's post (should work - mods can edit)
curl -X PUT http://localhost:8080/posts/1 \
  -H "Authorization: Bearer $MOD_TOKEN" \
  -d '{"title":"Edited by Mod","content":"Updated"}'

# Alice tries to edit Bob's post (should FAIL - not owner/mod/admin)
curl -X PUT http://localhost:8080/posts/2 \
  -H "Authorization: Bearer $USER_TOKEN" \
  -d '{"title":"Trying","content":"Should fail"}'
# Expected: 403 Forbidden
```

---

## What You've Learned

✅ **Complex RBAC patterns** - 4-tier permission system  
✅ **Resource ownership** - Checking permissions per resource  
✅ **Permission helpers** - Centralized authorization logic  
✅ **Optional authentication** - Gracefully handling guests  
✅ **Data filtering** - Showing different data per role  
✅ **Multi-role middleware** - Allowing multiple roles  
✅ **Business rules in code** - "Moderators can't publish"  
✅ **Production RBAC** - Real-world authorization patterns  

You now understand how to build complex authorization systems for production applications!
