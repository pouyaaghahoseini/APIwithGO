# Unit 3 - Exercise 2: Multi-Tenant Blog Platform with RBAC

**Difficulty**: Advanced  
**Estimated Time**: 75-90 minutes  
**Concepts Covered**: Role-based access control, resource ownership, multi-role authorization, permission systems

---

## Objective

Build a blog platform where users can create posts, but with different permission levels:
- **Guest**: Can read published posts only
- **User**: Can create/edit/delete their own posts
- **Moderator**: Can edit/delete any post, cannot publish
- **Admin**: Can do everything including publish any post

This exercise teaches complex authorization patterns beyond simple role checking.

---

## Requirements

### Data Models

```go
type User struct {
    ID           int       `json:"id"`
    Username     string    `json:"username"`
    Email        string    `json:"email"`
    PasswordHash string    `json:"-"`
    Role         string    `json:"role"` // "user", "moderator", "admin"
    CreatedAt    time.Time `json:"created_at"`
}

type Post struct {
    ID        int       `json:"id"`
    Title     string    `json:"title"`
    Content   string    `json:"content"`
    AuthorID  int       `json:"author_id"`
    Author    User      `json:"author"`  // Embedded for responses
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
```

### Permission Matrix

| Action | Guest | User | Moderator | Admin |
|--------|-------|------|-----------|-------|
| Read published posts | ✓ | ✓ | ✓ | ✓ |
| Read unpublished posts | ✗ | Own only | ✓ | ✓ |
| Create post | ✗ | ✓ | ✓ | ✓ |
| Edit own post | ✗ | ✓ | ✓ | ✓ |
| Edit others' posts | ✗ | ✗ | ✓ | ✓ |
| Delete own post | ✗ | ✓ | ✓ | ✓ |
| Delete others' posts | ✗ | ✗ | ✓ | ✓ |
| Publish own post | ✗ | ✓ | ✗ | ✓ |
| Publish others' posts | ✗ | ✗ | ✗ | ✓ |
| Approve comments | ✗ | ✗ | ✓ | ✓ |
| Delete comments | ✗ | Own only | ✓ | ✓ |

### API Endpoints

#### Authentication
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | /register | No | Register new user (default role: "user") |
| POST | /login | No | Login and get token |

#### Posts
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | /posts | Optional | List published posts (or all if admin/mod) |
| GET | /posts/{id} | Optional | Get single post |
| POST | /posts | Yes | Create post |
| PUT | /posts/{id} | Yes | Update post (check permissions) |
| DELETE | /posts/{id} | Yes | Delete post (check permissions) |
| POST | /posts/{id}/publish | Yes | Publish post (user: own, admin: any) |
| POST | /posts/{id}/unpublish | Yes | Unpublish post |

#### Comments
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | /posts/{id}/comments | Optional | Get approved comments |
| POST | /posts/{id}/comments | Yes | Add comment |
| DELETE | /comments/{id} | Yes | Delete comment (own or mod/admin) |
| POST | /comments/{id}/approve | Yes (Mod/Admin) | Approve comment |

#### Admin
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | /admin/users | Admin | List all users |
| PUT | /admin/users/{id}/role | Admin | Change user role |
| GET | /admin/stats | Admin | Get platform statistics |

### Request/Response Examples

**Create Post (POST /posts)**:
```json
Request (with Bearer token):
{
  "title": "My First Blog Post",
  "content": "This is the content of my first post..."
}

Response (201 Created):
{
  "id": 1,
  "title": "My First Blog Post",
  "content": "This is the content...",
  "author_id": 1,
  "author": {
    "id": 1,
    "username": "johndoe",
    "role": "user"
  },
  "published": false,
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

**Publish Post (POST /posts/1/publish)**:
```json
Response (200 OK):
{
  "id": 1,
  "title": "My First Blog Post",
  "content": "This is the content...",
  "published": true,
  "updated_at": "2024-01-15T10:35:00Z"
}
```

**Add Comment (POST /posts/1/comments)**:
```json
Request:
{
  "content": "Great post!"
}

Response (201 Created):
{
  "id": 1,
  "post_id": 1,
  "user_id": 2,
  "user": {
    "id": 2,
    "username": "jane",
    "role": "user"
  },
  "content": "Great post!",
  "approved": false,
  "created_at": "2024-01-15T10:40:00Z"
}
```

### Authorization Logic

**Can Edit Post?**
```
- Admin: YES (any post)
- Moderator: YES (any post)
- User: YES if author_id == user_id
- Guest: NO
```

**Can Publish Post?**
```
- Admin: YES (any post)
- Moderator: NO
- User: YES if author_id == user_id
- Guest: NO
```

**Can Delete Post?**
```
- Admin: YES (any post)
- Moderator: YES (any post)
- User: YES if author_id == user_id
- Guest: NO
```

**Can Approve Comment?**
```
- Admin: YES
- Moderator: YES
- User: NO
- Guest: NO
```

---

## Starter Code

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "sync"
    "time"
    
    "github.com/golang-jwt/jwt/v5"
    "github.com/gorilla/mux"
    "golang.org/x/crypto/bcrypt"
)

var jwtSecret = []byte("your-secret-key")

// Models
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

// Storage
var (
    users        = make(map[int]User)
    posts        = make(map[int]Post)
    comments     = make(map[int]Comment)
    usernames    = make(map[string]int)
    nextUserID   = 1
    nextPostID   = 1
    nextCommentID = 1
    usersMu      sync.RWMutex
    postsMu      sync.RWMutex
    commentsMu   sync.RWMutex
)

type contextKey string
const UserContextKey contextKey = "user"

// TODO: Implement authentication handlers
func register(w http.ResponseWriter, r *http.Request) {}
func login(w http.ResponseWriter, r *http.Request) {}

// TODO: Implement post handlers
func getPosts(w http.ResponseWriter, r *http.Request) {
    // If no auth or guest: show only published
    // If user: show published + own unpublished
    // If mod/admin: show all
}

func getPost(w http.ResponseWriter, r *http.Request) {
    // Check if user can view (published or owns or is mod/admin)
}

func createPost(w http.ResponseWriter, r *http.Request) {
    // Requires authentication
}

func updatePost(w http.ResponseWriter, r *http.Request) {
    // Check: canEdit(user, post)
}

func deletePost(w http.ResponseWriter, r *http.Request) {
    // Check: canDelete(user, post)
}

func publishPost(w http.ResponseWriter, r *http.Request) {
    // Check: canPublish(user, post)
}

func unpublishPost(w http.ResponseWriter, r *http.Request) {
    // Check: canPublish(user, post)
}

// TODO: Implement comment handlers
func getComments(w http.ResponseWriter, r *http.Request) {
    // Show only approved comments (unless mod/admin)
}

func createComment(w http.ResponseWriter, r *http.Request) {
    // Requires authentication
}

func deleteComment(w http.ResponseWriter, r *http.Request) {
    // Check: own comment OR mod/admin
}

func approveComment(w http.ResponseWriter, r *http.Request) {
    // Requires mod or admin role
}

// TODO: Implement admin handlers
func getAllUsers(w http.ResponseWriter, r *http.Request) {
    // Admin only
}

func changeUserRole(w http.ResponseWriter, r *http.Request) {
    // Admin only
}

func getStats(w http.ResponseWriter, r *http.Request) {
    // Admin only - return total users, posts, comments
}

// TODO: Implement permission helpers
func canEdit(user *Claims, post Post) bool {
    // Admin/Mod: YES
    // User: if owns
}

func canDelete(user *Claims, post Post) bool {
    // Admin/Mod: YES
    // User: if owns
}

func canPublish(user *Claims, post Post) bool {
    // Admin: YES
    // User: if owns
    // Mod: NO
}

func canApprove(user *Claims) bool {
    // Admin/Mod: YES
}

// TODO: Implement middleware
func optionalAuthMiddleware(next http.Handler) http.Handler {
    // Extract token if present, but don't require it
    // Add user to context if valid token
}

func requireAuthMiddleware(next http.Handler) http.Handler {
    // Require valid token
}

func requireRole(roles ...string) func(http.Handler) http.Handler {
    // Check user has one of the allowed roles
}

// Helpers
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}

func getUserFromContext(r *http.Request) (*Claims, bool) {
    user, ok := r.Context().Value(UserContextKey).(*Claims)
    return user, ok
}

func main() {
    r := mux.NewRouter()
    
    // TODO: Register routes with appropriate middleware
    
    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", r)
}
```

---

## Testing Scenarios

### Setup: Create Users with Different Roles

```bash
# Register regular user
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","email":"alice@test.com","password":"Password123"}'

# Save token as USER_TOKEN

# Register another user (will be promoted to moderator)
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{"username":"bob","email":"bob@test.com","password":"Password123"}'

# Register admin (will be promoted to admin)
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","email":"admin@test.com","password":"Password123"}'

# Save token as ADMIN_TOKEN

# Promote bob to moderator (using admin token)
curl -X PUT http://localhost:8080/admin/users/2/role \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"role":"moderator"}'

# Promote admin to admin role
curl -X PUT http://localhost:8080/admin/users/3/role \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"role":"admin"}'
```

### Test Post Permissions

```bash
# Alice creates a post
curl -X POST http://localhost:8080/posts \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"Alice Post","content":"Content by Alice"}'

# Alice publishes her own post (should work)
curl -X POST http://localhost:8080/posts/1/publish \
  -H "Authorization: Bearer $USER_TOKEN"

# Bob (moderator) creates and tries to publish (publish should fail)
curl -X POST http://localhost:8080/posts \
  -H "Authorization: Bearer $MOD_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"Bob Post","content":"Content by Bob"}'

curl -X POST http://localhost:8080/posts/2/publish \
  -H "Authorization: Bearer $MOD_TOKEN"
# Should return 403 Forbidden

# Admin publishes Bob's post (should work)
curl -X POST http://localhost:8080/posts/2/publish \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Bob (moderator) edits Alice's post (should work)
curl -X PUT http://localhost:8080/posts/1 \
  -H "Authorization: Bearer $MOD_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"Edited by Moderator","content":"Updated content"}'

# Alice tries to edit Bob's post (should fail)
curl -X PUT http://localhost:8080/posts/2 \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"Trying to edit","content":"Should fail"}'
```

### Test Comment Permissions

```bash
# Alice adds comment (requires approval)
curl -X POST http://localhost:8080/posts/1/comments \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content":"Great post!"}'

# Regular user tries to approve (should fail)
curl -X POST http://localhost:8080/comments/1/approve \
  -H "Authorization: Bearer $USER_TOKEN"

# Moderator approves comment (should work)
curl -X POST http://localhost:8080/comments/1/approve \
  -H "Authorization: Bearer $MOD_TOKEN"
```

### Test Guest Access

```bash
# View published posts (no auth)
curl http://localhost:8080/posts

# Try to create post without auth (should fail)
curl -X POST http://localhost:8080/posts \
  -H "Content-Type: application/json" \
  -d '{"title":"Test","content":"Test"}'
```

---

## Bonus Challenges

### Bonus 1: Post Draft System
Add draft status separate from published:
- `status`: "draft", "pending", "published"
- Users can submit for review (draft → pending)
- Moderators can approve (pending → published)

### Bonus 2: Comment Moderation Queue
GET /admin/comments/pending - show unapproved comments

### Bonus 3: Activity Log
Log all moderation actions:
- Who edited/deleted what
- Who approved comments
- Role changes

### Bonus 4: Bulk Operations
POST /admin/posts/bulk-publish - publish multiple posts
DELETE /admin/comments/spam - delete multiple comments

### Bonus 5: User Suspension
Admins can suspend users (suspended users can't create content)

---

## Hints

### Hint 1: Optional Auth Middleware
```go
func optionalAuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            next.ServeHTTP(w, r)  // No auth, continue without user
            return
        }
        
        // Try to validate token
        parts := strings.Split(authHeader, " ")
        if len(parts) == 2 && parts[0] == "Bearer" {
            claims, err := validateToken(parts[1])
            if err == nil {
                ctx := context.WithValue(r.Context(), UserContextKey, claims)
                next.ServeHTTP(w, r.WithContext(ctx))
                return
            }
        }
        
        // Invalid token, but don't block request
        next.ServeHTTP(w, r)
    })
}
```

### Hint 2: Permission Check Functions
```go
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

func canPublish(user *Claims, post Post) bool {
    if user == nil {
        return false
    }
    
    // Admins can publish anything
    if user.Role == "admin" {
        return true
    }
    
    // Moderators CANNOT publish
    if user.Role == "moderator" {
        return false
    }
    
    // Users can publish their own posts
    return user.UserID == post.AuthorID
}
```

### Hint 3: Filtering Posts by Permission
```go
func getPosts(w http.ResponseWriter, r *http.Request) {
    user, authenticated := getUserFromContext(r)
    
    postsMu.RLock()
    usersMu.RLock()
    defer postsMu.RUnlock()
    defer usersMu.RUnlock()
    
    var results []Post
    for _, post := range posts {
        // Admin/Mod see everything
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
```

---

## What You're Learning

✅ Complex role-based authorization  
✅ Resource ownership permissions  
✅ Multi-level access control  
✅ Permission helper functions  
✅ Optional vs required authentication  
✅ Filtering data by permissions  
✅ Real-world authorization patterns  
✅ Moderation workflows  

This exercise demonstrates production-grade authorization systems!
