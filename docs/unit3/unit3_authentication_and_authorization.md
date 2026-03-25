# Unit 3: Authentication & Authorization

**Duration**: 90 minutes  
**Prerequisites**: Unit 1 (Go fundamentals), Unit 2 (HTTP servers)  
**Goal**: Implement secure authentication with JWT and role-based authorization

---

## 3.1 Authentication vs Authorization

**Authentication**: Who are you? (Identity verification)
- Login with username/password
- JWT tokens
- API keys
- OAuth

**Authorization**: What can you do? (Permission verification)
- Role-based access control (RBAC)
- User roles (admin, user, guest)
- Resource-level permissions

---

## 3.2 Password Security

### Never Store Plain Text Passwords!

```go
// WRONG - NEVER DO THIS
user.Password = request.Password  // Stored as plain text!

// RIGHT - Always hash passwords
hashedPassword, err := bcrypt.GenerateFromPassword([]byte(request.Password), bcrypt.DefaultCost)
user.Password = string(hashedPassword)
```

### Using bcrypt for Password Hashing

**Install bcrypt**:
```bash
go get golang.org/x/crypto/bcrypt
```

**Hashing a password**:
```go
import "golang.org/x/crypto/bcrypt"

func hashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(bytes), err
}

// Usage
hashedPassword, err := hashPassword("mySecretPassword123")
if err != nil {
    // Handle error
}
// Store hashedPassword in database
```

**Verifying a password**:
```go
func checkPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}

// Usage
if checkPassword(providedPassword, storedHash) {
    // Password is correct
} else {
    // Invalid password
}
```

**Why bcrypt?**
- Slow by design (prevents brute force)
- Automatically includes salt
- Industry standard
- Configurable cost factor

---

## 3.3 JSON Web Tokens (JWT)

JWT is a standard for securely transmitting information between parties as a JSON object.

### JWT Structure

A JWT consists of three parts separated by dots:
```
header.payload.signature
```

Example:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoxMjMsInVzZXJuYW1lIjoiam9obiIsImV4cCI6MTY0MDk5NTIwMH0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**Header** (red):
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload** (purple):
```json
{
  "user_id": 123,
  "username": "john",
  "role": "admin",
  "exp": 1640995200
}
```

**Signature** (blue):
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

### Installing JWT Library

```bash
go get github.com/golang-jwt/jwt/v5
```

### Creating JWTs

```go
package main

import (
    "time"
    
    "github.com/golang-jwt/jwt/v5"
)

// Secret key - should be in environment variable
var jwtSecret = []byte("your-secret-key-change-this")

// Claims structure
type Claims struct {
    UserID   int    `json:"user_id"`
    Username string `json:"username"`
    Role     string `json:"role"`
    jwt.RegisteredClaims
}

func generateToken(userID int, username, role string) (string, error) {
    // Create claims
    claims := Claims{
        UserID:   userID,
        Username: username,
        Role:     role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "my-app",
        },
    }
    
    // Create token with claims
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    
    // Sign token with secret
    tokenString, err := token.SignedString(jwtSecret)
    if err != nil {
        return "", err
    }
    
    return tokenString, nil
}
```

### Validating JWTs

```go
func validateToken(tokenString string) (*Claims, error) {
    // Parse token
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        // Verify signing method
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        return jwtSecret, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    // Extract claims
    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }
    
    return nil, fmt.Errorf("invalid token")
}
```

---

## 3.4 User Registration

```go
type User struct {
    ID           int       `json:"id"`
    Username     string    `json:"username"`
    Email        string    `json:"email"`
    PasswordHash string    `json:"-"`  // Never send password in JSON
    Role         string    `json:"role"`
    CreatedAt    time.Time `json:"created_at"`
}

type RegisterRequest struct {
    Username string `json:"username"`
    Email    string `json:"email"`
    Password string `json:"password"`
}

type RegisterResponse struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
    Token    string `json:"token"`
}

func registerHandler(w http.ResponseWriter, r *http.Request) {
    var req RegisterRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()
    
    // Validate input
    if len(req.Username) < 3 {
        respondError(w, http.StatusBadRequest, "Username must be at least 3 characters")
        return
    }
    
    if len(req.Password) < 8 {
        respondError(w, http.StatusBadRequest, "Password must be at least 8 characters")
        return
    }
    
    // Check if username exists
    if userExists(req.Username) {
        respondError(w, http.StatusConflict, "Username already exists")
        return
    }
    
    // Hash password
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
    if err != nil {
        respondError(w, http.StatusInternalServerError, "Failed to hash password")
        return
    }
    
    // Create user
    user := User{
        ID:           nextUserID,
        Username:     req.Username,
        Email:        req.Email,
        PasswordHash: string(hashedPassword),
        Role:         "user",  // Default role
        CreatedAt:    time.Now(),
    }
    
    // Save user (to database in real app)
    users[nextUserID] = user
    nextUserID++
    
    // Generate JWT token
    token, err := generateToken(user.ID, user.Username, user.Role)
    if err != nil {
        respondError(w, http.StatusInternalServerError, "Failed to generate token")
        return
    }
    
    // Return user info with token
    response := RegisterResponse{
        ID:       user.ID,
        Username: user.Username,
        Email:    user.Email,
        Token:    token,
    }
    
    respondJSON(w, http.StatusCreated, response)
}
```

---

## 3.5 User Login

```go
type LoginRequest struct {
    Username string `json:"username"`
    Password string `json:"password"`
}

type LoginResponse struct {
    Token string `json:"token"`
    User  User   `json:"user"`
}

func loginHandler(w http.ResponseWriter, r *http.Request) {
    var req LoginRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()
    
    // Find user by username
    user, found := findUserByUsername(req.Username)
    if !found {
        respondError(w, http.StatusUnauthorized, "Invalid credentials")
        return
    }
    
    // Verify password
    err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(req.Password))
    if err != nil {
        respondError(w, http.StatusUnauthorized, "Invalid credentials")
        return
    }
    
    // Generate token
    token, err := generateToken(user.ID, user.Username, user.Role)
    if err != nil {
        respondError(w, http.StatusInternalServerError, "Failed to generate token")
        return
    }
    
    // Return token and user info
    response := LoginResponse{
        Token: token,
        User:  user,
    }
    
    respondJSON(w, http.StatusOK, response)
}
```

**Security Note**: Always return the same error message for "user not found" and "wrong password" to prevent username enumeration attacks.

---

## 3.6 Authentication Middleware

Middleware that validates JWT tokens and adds user info to request context.

```go
package middleware

import (
    "context"
    "net/http"
    "strings"
)

type contextKey string

const UserContextKey contextKey = "user"

func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Extract token from Authorization header
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            http.Error(w, "Authorization header required", http.StatusUnauthorized)
            return
        }
        
        // Expected format: "Bearer <token>"
        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            http.Error(w, "Invalid authorization header format", http.StatusUnauthorized)
            return
        }
        
        tokenString := parts[1]
        
        // Validate token
        claims, err := validateToken(tokenString)
        if err != nil {
            http.Error(w, "Invalid or expired token", http.StatusUnauthorized)
            return
        }
        
        // Add claims to request context
        ctx := context.WithValue(r.Context(), UserContextKey, claims)
        
        // Call next handler with updated context
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Helper to extract user from context
func GetUserFromContext(r *http.Request) (*Claims, bool) {
    user, ok := r.Context().Value(UserContextKey).(*Claims)
    return user, ok
}
```

### Using Auth Middleware

```go
func main() {
    r := mux.NewRouter()
    
    // Public routes
    r.HandleFunc("/register", registerHandler).Methods("POST")
    r.HandleFunc("/login", loginHandler).Methods("POST")
    
    // Protected routes
    protected := r.PathPrefix("/api").Subrouter()
    protected.Use(middleware.AuthMiddleware)
    
    protected.HandleFunc("/profile", getProfileHandler).Methods("GET")
    protected.HandleFunc("/posts", createPostHandler).Methods("POST")
    protected.HandleFunc("/posts/{id}", deletePostHandler).Methods("DELETE")
    
    http.ListenAndServe(":8080", r)
}
```

### Accessing User in Protected Routes

```go
func getProfileHandler(w http.ResponseWriter, r *http.Request) {
    // Get authenticated user from context
    user, ok := middleware.GetUserFromContext(r)
    if !ok {
        respondError(w, http.StatusUnauthorized, "User not found in context")
        return
    }
    
    // Use user information
    fmt.Printf("User ID: %d, Username: %s, Role: %s\n", 
        user.UserID, user.Username, user.Role)
    
    // Fetch full user details from database
    fullUser := getUserByID(user.UserID)
    
    respondJSON(w, http.StatusOK, fullUser)
}
```

---

## 3.7 Role-Based Authorization

Middleware to restrict access based on user roles.

```go
func RequireRole(allowedRoles ...string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Get user from context
            user, ok := GetUserFromContext(r)
            if !ok {
                http.Error(w, "Unauthorized", http.StatusUnauthorized)
                return
            }
            
            // Check if user has required role
            hasRole := false
            for _, role := range allowedRoles {
                if user.Role == role {
                    hasRole = true
                    break
                }
            }
            
            if !hasRole {
                http.Error(w, "Forbidden - insufficient permissions", http.StatusForbidden)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}
```

### Using Role Middleware

```go
func main() {
    r := mux.NewRouter()
    
    // Public routes
    r.HandleFunc("/register", registerHandler).Methods("POST")
    r.HandleFunc("/login", loginHandler).Methods("POST")
    
    // User routes (authenticated)
    userRoutes := r.PathPrefix("/api").Subrouter()
    userRoutes.Use(AuthMiddleware)
    userRoutes.HandleFunc("/profile", getProfile).Methods("GET")
    userRoutes.HandleFunc("/posts", getPosts).Methods("GET")
    
    // Admin routes (authenticated + admin role)
    adminRoutes := r.PathPrefix("/api/admin").Subrouter()
    adminRoutes.Use(AuthMiddleware)
    adminRoutes.Use(RequireRole("admin"))
    adminRoutes.HandleFunc("/users", getAllUsers).Methods("GET")
    adminRoutes.HandleFunc("/users/{id}", deleteUser).Methods("DELETE")
    adminRoutes.HandleFunc("/stats", getStats).Methods("GET")
    
    // Multiple roles allowed
    moderatorRoutes := r.PathPrefix("/api/moderate").Subrouter()
    moderatorRoutes.Use(AuthMiddleware)
    moderatorRoutes.Use(RequireRole("admin", "moderator"))
    moderatorRoutes.HandleFunc("/posts/{id}", approvePost).Methods("POST")
    
    http.ListenAndServe(":8080", r)
}
```

---

## 3.8 Complete Authentication System Example

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "strings"
    "sync"
    "time"
    
    "github.com/golang-jwt/jwt/v5"
    "github.com/gorilla/mux"
    "golang.org/x/crypto/bcrypt"
)

// JWT secret
var jwtSecret = []byte("your-secret-key-change-in-production")

// Models
type User struct {
    ID           int       `json:"id"`
    Username     string    `json:"username"`
    Email        string    `json:"email"`
    PasswordHash string    `json:"-"`
    Role         string    `json:"role"`
    CreatedAt    time.Time `json:"created_at"`
}

type Claims struct {
    UserID   int    `json:"user_id"`
    Username string `json:"username"`
    Role     string `json:"role"`
    jwt.RegisteredClaims
}

// Storage
var (
    users      = make(map[int]User)
    usernames  = make(map[string]int) // username -> user ID
    nextUserID = 1
    usersMu    sync.RWMutex
)

// Context key
type contextKey string
const UserContextKey contextKey = "user"

// Register
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
    
    // Validate
    if len(req.Username) < 3 {
        respondError(w, http.StatusBadRequest, "Username must be at least 3 characters")
        return
    }
    if len(req.Password) < 8 {
        respondError(w, http.StatusBadRequest, "Password must be at least 8 characters")
        return
    }
    
    // Check if username exists
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
        Role:         "user",
        CreatedAt:    time.Now(),
    }
    users[nextUserID] = user
    usernames[req.Username] = nextUserID
    nextUserID++
    usersMu.Unlock()
    
    // Generate token
    token, err := generateToken(user.ID, user.Username, user.Role)
    if err != nil {
        respondError(w, http.StatusInternalServerError, "Failed to generate token")
        return
    }
    
    respondJSON(w, http.StatusCreated, map[string]interface{}{
        "user":  user,
        "token": token,
    })
}

// Login
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
    token, err := generateToken(user.ID, user.Username, user.Role)
    if err != nil {
        respondError(w, http.StatusInternalServerError, "Failed to generate token")
        return
    }
    
    respondJSON(w, http.StatusOK, map[string]interface{}{
        "user":  user,
        "token": token,
    })
}

// Protected route example
func getProfile(w http.ResponseWriter, r *http.Request) {
    user, _ := r.Context().Value(UserContextKey).(*Claims)
    
    usersMu.RLock()
    fullUser := users[user.UserID]
    usersMu.RUnlock()
    
    respondJSON(w, http.StatusOK, fullUser)
}

// Admin-only route example
func getAllUsers(w http.ResponseWriter, r *http.Request) {
    usersMu.RLock()
    userList := make([]User, 0, len(users))
    for _, user := range users {
        userList = append(userList, user)
    }
    usersMu.RUnlock()
    
    respondJSON(w, http.StatusOK, userList)
}

// JWT functions
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

// Middleware
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            respondError(w, http.StatusUnauthorized, "Authorization header required")
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

func requireRole(allowedRoles ...string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            user, ok := r.Context().Value(UserContextKey).(*Claims)
            if !ok {
                respondError(w, http.StatusUnauthorized, "Unauthorized")
                return
            }
            
            hasRole := false
            for _, role := range allowedRoles {
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

// Helpers
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}

func main() {
    r := mux.NewRouter()
    
    // Public routes
    r.HandleFunc("/register", register).Methods("POST")
    r.HandleFunc("/login", login).Methods("POST")
    
    // Protected routes
    api := r.PathPrefix("/api").Subrouter()
    api.Use(authMiddleware)
    api.HandleFunc("/profile", getProfile).Methods("GET")
    
    // Admin routes
    admin := r.PathPrefix("/api/admin").Subrouter()
    admin.Use(authMiddleware)
    admin.Use(requireRole("admin"))
    admin.HandleFunc("/users", getAllUsers).Methods("GET")
    
    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", r)
}
```

---

## 3.9 Testing Authentication

### Register a User
```bash
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "john",
    "email": "john@example.com",
    "password": "password123"
  }'

# Response:
# {
#   "user": {
#     "id": 1,
#     "username": "john",
#     "email": "john@example.com",
#     "role": "user",
#     "created_at": "2024-01-15T10:30:00Z"
#   },
#   "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
# }
```

### Login
```bash
curl -X POST http://localhost:8080/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "john",
    "password": "password123"
  }'
```

### Access Protected Route
```bash
# Save token from login/register
TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

# Use token in Authorization header
curl http://localhost:8080/api/profile \
  -H "Authorization: Bearer $TOKEN"
```

### Test Authorization (403 Forbidden)
```bash
# Regular user trying to access admin route
curl http://localhost:8080/api/admin/users \
  -H "Authorization: Bearer $TOKEN"
  
# Response: 403 Forbidden
```

---

## 3.10 Security Best Practices

### 1. Store Secrets Securely
```go
// WRONG
var jwtSecret = []byte("my-secret-key")

// RIGHT - Use environment variables
var jwtSecret = []byte(os.Getenv("JWT_SECRET"))
```

### 2. Use HTTPS in Production
Never send tokens over plain HTTP in production. Always use HTTPS.

### 3. Token Expiration
```go
ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour))
```
- Short expiration for sensitive operations (15 min - 1 hour)
- Longer for general use (24 hours)
- Implement refresh tokens for better UX

### 4. Rate Limiting
Prevent brute force attacks on login endpoint.

### 5. Password Requirements
- Minimum 8 characters
- Mix of uppercase, lowercase, numbers, symbols
- Check against common password lists

### 6. Timing Attack Prevention
```go
// Always use bcrypt.CompareHashAndPassword
// It's designed to take constant time
err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
```

### 7. Don't Leak Information
```go
// WRONG - Reveals which part is wrong
if !userExists {
    return "User not found"
} else if !passwordCorrect {
    return "Wrong password"
}

// RIGHT - Same message for both
return "Invalid credentials"
```

---

## Key Takeaways

✅ **Passwords**: Always hash with bcrypt, never store plain text  
✅ **JWT**: Stateless authentication with signed tokens  
✅ **Claims**: Store user info in token payload  
✅ **Middleware**: Validate tokens and add user to context  
✅ **Authorization**: Role-based access control with middleware  
✅ **Security**: HTTPS, secrets in env vars, rate limiting  
✅ **Error Messages**: Don't leak information about users  

---

## What's Next?

Unit 4 will cover API versioning strategies to evolve your API without breaking existing clients. You now have a secure authentication system ready for production! 🔒
