# Unit 3 - Exercise 1 Solution: Secure User Management API

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
    "strings"
    "sync"
    "time"

    "github.com/golang-jwt/jwt/v5"
    "github.com/gorilla/mux"
    "golang.org/x/crypto/bcrypt"
)

var jwtSecret = []byte("change-this-secret-key-in-production")

// Models
type User struct {
    ID           int       `json:"id"`
    Username     string    `json:"username"`
    Email        string    `json:"email"`
    PasswordHash string    `json:"-"`
    FullName     string    `json:"full_name"`
    Bio          string    `json:"bio"`
    Role         string    `json:"role"`
    CreatedAt    time.Time `json:"created_at"`
    UpdatedAt    time.Time `json:"updated_at"`
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
    usernames  = make(map[string]int)
    emails     = make(map[string]int)
    nextUserID = 1
    usersMu    sync.RWMutex
)

type contextKey string

const UserContextKey contextKey = "user"

// =============================================================================
// AUTHENTICATION HANDLERS
// =============================================================================

func registerHandler(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Username string `json:"username"`
        Email    string `json:"email"`
        Password string `json:"password"`
        FullName string `json:"full_name"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Validate username
    if err := validateUsername(req.Username); err != nil {
        respondError(w, http.StatusBadRequest, err.Error())
        return
    }

    // Validate email
    if err := validateEmail(req.Email); err != nil {
        respondError(w, http.StatusBadRequest, err.Error())
        return
    }

    // Validate password
    if err := validatePassword(req.Password); err != nil {
        respondError(w, http.StatusBadRequest, err.Error())
        return
    }

    // Validate full name
    if req.FullName != "" && (len(req.FullName) < 2 || len(req.FullName) > 50) {
        respondError(w, http.StatusBadRequest, "Full name must be between 2 and 50 characters")
        return
    }

    // Check uniqueness
    usersMu.RLock()
    _, usernameExists := usernames[req.Username]
    _, emailExists := emails[req.Email]
    usersMu.RUnlock()

    if usernameExists {
        respondError(w, http.StatusConflict, "Username already exists")
        return
    }

    if emailExists {
        respondError(w, http.StatusConflict, "Email already exists")
        return
    }

    // Hash password
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
    if err != nil {
        respondError(w, http.StatusInternalServerError, "Failed to process password")
        return
    }

    // Create user
    now := time.Now()
    usersMu.Lock()
    user := User{
        ID:           nextUserID,
        Username:     req.Username,
        Email:        req.Email,
        PasswordHash: string(hashedPassword),
        FullName:     req.FullName,
        Bio:          "",
        Role:         "user",
        CreatedAt:    now,
        UpdatedAt:    now,
    }
    users[nextUserID] = user
    usernames[req.Username] = nextUserID
    emails[req.Email] = nextUserID
    nextUserID++
    usersMu.Unlock()

    // Generate token
    token, err := generateToken(user.ID, user.Username, user.Role)
    if err != nil {
        respondError(w, http.StatusInternalServerError, "Failed to generate token")
        return
    }

    // Return response
    respondJSON(w, http.StatusCreated, map[string]interface{}{
        "user":  user,
        "token": token,
    })
}

func loginHandler(w http.ResponseWriter, r *http.Request) {
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
        "token": token,
        "user":  user,
    })
}

// =============================================================================
// PROFILE HANDLERS
// =============================================================================

func getProfileHandler(w http.ResponseWriter, r *http.Request) {
    claims, ok := getUserFromContext(r)
    if !ok {
        respondError(w, http.StatusUnauthorized, "Unauthorized")
        return
    }

    usersMu.RLock()
    user, exists := users[claims.UserID]
    usersMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "User not found")
        return
    }

    respondJSON(w, http.StatusOK, user)
}

func updateProfileHandler(w http.ResponseWriter, r *http.Request) {
    claims, ok := getUserFromContext(r)
    if !ok {
        respondError(w, http.StatusUnauthorized, "Unauthorized")
        return
    }

    var req struct {
        FullName string `json:"full_name"`
        Bio      string `json:"bio"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Validate
    if req.FullName != "" && (len(req.FullName) < 2 || len(req.FullName) > 50) {
        respondError(w, http.StatusBadRequest, "Full name must be between 2 and 50 characters")
        return
    }

    if len(req.Bio) > 500 {
        respondError(w, http.StatusBadRequest, "Bio must be 500 characters or less")
        return
    }

    // Update user
    usersMu.Lock()
    user, exists := users[claims.UserID]
    if !exists {
        usersMu.Unlock()
        respondError(w, http.StatusNotFound, "User not found")
        return
    }

    user.FullName = req.FullName
    user.Bio = req.Bio
    user.UpdatedAt = time.Now()
    users[claims.UserID] = user
    usersMu.Unlock()

    respondJSON(w, http.StatusOK, user)
}

func changePasswordHandler(w http.ResponseWriter, r *http.Request) {
    claims, ok := getUserFromContext(r)
    if !ok {
        respondError(w, http.StatusUnauthorized, "Unauthorized")
        return
    }

    var req struct {
        CurrentPassword string `json:"current_password"`
        NewPassword     string `json:"new_password"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Get user
    usersMu.RLock()
    user, exists := users[claims.UserID]
    usersMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "User not found")
        return
    }

    // Verify current password
    err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(req.CurrentPassword))
    if err != nil {
        respondError(w, http.StatusUnauthorized, "Current password is incorrect")
        return
    }

    // Validate new password
    if err := validatePassword(req.NewPassword); err != nil {
        respondError(w, http.StatusBadRequest, err.Error())
        return
    }

    // Check new password is different
    if req.CurrentPassword == req.NewPassword {
        respondError(w, http.StatusBadRequest, "New password must be different from current password")
        return
    }

    // Hash new password
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(req.NewPassword), bcrypt.DefaultCost)
    if err != nil {
        respondError(w, http.StatusInternalServerError, "Failed to process password")
        return
    }

    // Update password
    usersMu.Lock()
    user.PasswordHash = string(hashedPassword)
    user.UpdatedAt = time.Now()
    users[claims.UserID] = user
    usersMu.Unlock()

    respondJSON(w, http.StatusOK, map[string]string{
        "message": "Password updated successfully",
    })
}

func deleteAccountHandler(w http.ResponseWriter, r *http.Request) {
    claims, ok := getUserFromContext(r)
    if !ok {
        respondError(w, http.StatusUnauthorized, "Unauthorized")
        return
    }

    usersMu.Lock()
    user, exists := users[claims.UserID]
    if !exists {
        usersMu.Unlock()
        respondError(w, http.StatusNotFound, "User not found")
        return
    }

    // Delete user
    delete(users, claims.UserID)
    delete(usernames, user.Username)
    delete(emails, user.Email)
    usersMu.Unlock()

    w.WriteHeader(http.StatusNoContent)
}

// =============================================================================
// VALIDATION FUNCTIONS
// =============================================================================

func validateUsername(username string) error {
    username = strings.TrimSpace(username)

    if len(username) < 3 || len(username) > 20 {
        return errors.New("Username must be between 3 and 20 characters")
    }

    // Alphanumeric and underscores only
    validUsername := regexp.MustCompile(`^[a-zA-Z0-9_]+$`)
    if !validUsername.MatchString(username) {
        return errors.New("Username can only contain letters, numbers, and underscores")
    }

    return nil
}

func validateEmail(email string) error {
    email = strings.TrimSpace(email)

    if email == "" {
        return errors.New("Email is required")
    }

    // Simple email regex
    emailRegex := regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)
    if !emailRegex.MatchString(email) {
        return errors.New("Invalid email format")
    }

    return nil
}

func validatePassword(password string) error {
    if len(password) < 8 {
        return errors.New("Password must be at least 8 characters")
    }

    hasUpper := regexp.MustCompile(`[A-Z]`).MatchString(password)
    hasLower := regexp.MustCompile(`[a-z]`).MatchString(password)
    hasNumber := regexp.MustCompile(`[0-9]`).MatchString(password)

    if !hasUpper || !hasLower || !hasNumber {
        return errors.New("Password must contain uppercase, lowercase, and number")
    }

    return nil
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
        // Verify signing method
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        return jwtSecret, nil
    })

    if err != nil {
        return nil, err
    }

    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }

    return nil, errors.New("invalid token")
}

// =============================================================================
// MIDDLEWARE
// =============================================================================

func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            respondError(w, http.StatusUnauthorized, "Authorization header required")
            return
        }

        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            respondError(w, http.StatusUnauthorized, "Invalid authorization header format")
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
    r.HandleFunc("/register", registerHandler).Methods("POST")
    r.HandleFunc("/login", loginHandler).Methods("POST")

    // Protected routes
    api := r.PathPrefix("/api").Subrouter()
    api.Use(authMiddleware)

    api.HandleFunc("/profile", getProfileHandler).Methods("GET")
    api.HandleFunc("/profile", updateProfileHandler).Methods("PUT")
    api.HandleFunc("/password", changePasswordHandler).Methods("PUT")
    api.HandleFunc("/account", deleteAccountHandler).Methods("DELETE")

    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", r)
}
```

---

## Key Concepts Explained

### 1. Password Hashing with bcrypt

```go
// Hashing (registration)
hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)

// Verification (login)
err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
if err != nil {
    // Password incorrect
}
```

**Why bcrypt?**
- Slow by design (prevents brute force)
- Automatically includes salt
- Industry standard
- `CompareHashAndPassword` is constant-time (prevents timing attacks)

### 2. JWT Token Structure

```go
type Claims struct {
    UserID   int    `json:"user_id"`
    Username string `json:"username"`
    Role     string `json:"role"`
    jwt.RegisteredClaims  // Includes exp, iat, etc.
}
```

**Token contains**:
- User identification (ID, username)
- Authorization info (role)
- Expiration time
- Issued at time

### 3. Validation Patterns

```go
func validatePassword(password string) error {
    if len(password) < 8 {
        return errors.New("Too short")
    }
    
    hasUpper := regexp.MustCompile(`[A-Z]`).MatchString(password)
    hasLower := regexp.MustCompile(`[a-z]`).MatchString(password)
    hasNumber := regexp.MustCompile(`[0-9]`).MatchString(password)
    
    if !hasUpper || !hasLower || !hasNumber {
        return errors.New("Missing required character types")
    }
    
    return nil
}
```

**Why strict validation?**
- Prevents weak passwords
- Enforces security policy
- Returns clear error messages

### 4. Context for Request Data

```go
// Store in middleware
ctx := context.WithValue(r.Context(), UserContextKey, claims)
next.ServeHTTP(w, r.WithContext(ctx))

// Retrieve in handler
claims, ok := r.Context().Value(UserContextKey).(*Claims)
```

**Benefits**:
- Thread-safe request-scoped data
- No global state
- Clean handler signatures

### 5. Security: Don't Leak Information

```go
// WRONG - Reveals which is incorrect
if !userExists {
    return "User not found"
} else if !passwordCorrect {
    return "Wrong password"
}

// RIGHT - Same message for both
return "Invalid credentials"
```

This prevents username enumeration attacks.

---

## Bonus Solutions

### Bonus 1: Email Login

```go
func loginHandler(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Login    string `json:"login"`  // Can be username or email
        Password string `json:"password"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Try to find user by username or email
    usersMu.RLock()
    var user User
    var exists bool

    // Try username first
    if userID, ok := usernames[req.Login]; ok {
        user = users[userID]
        exists = true
    } else if userID, ok := emails[req.Login]; ok {
        // Try email
        user = users[userID]
        exists = true
    }
    usersMu.RUnlock()

    if !exists {
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

    respondJSON(w, http.StatusOK, map[string]interface{}{
        "token": token,
        "user":  user,
    })
}
```

### Bonus 2: Refresh Tokens

```go
type RefreshToken struct {
    Token     string
    UserID    int
    ExpiresAt time.Time
}

var (
    refreshTokens = make(map[string]RefreshToken)
    refreshMu     sync.RWMutex
)

func generateRefreshToken(userID int) (string, error) {
    // Generate random token
    token := uuid.New().String()
    
    refreshMu.Lock()
    refreshTokens[token] = RefreshToken{
        Token:     token,
        UserID:    userID,
        ExpiresAt: time.Now().Add(7 * 24 * time.Hour),
    }
    refreshMu.Unlock()
    
    return token, nil
}

func refreshHandler(w http.ResponseWriter, r *http.Request) {
    var req struct {
        RefreshToken string `json:"refresh_token"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }

    // Validate refresh token
    refreshMu.RLock()
    rt, exists := refreshTokens[req.RefreshToken]
    refreshMu.RUnlock()

    if !exists {
        respondError(w, http.StatusUnauthorized, "Invalid refresh token")
        return
    }

    if time.Now().After(rt.ExpiresAt) {
        respondError(w, http.StatusUnauthorized, "Refresh token expired")
        return
    }

    // Get user
    usersMu.RLock()
    user := users[rt.UserID]
    usersMu.RUnlock()

    // Generate new access token
    accessToken, err := generateToken(user.ID, user.Username, user.Role)
    if err != nil {
        respondError(w, http.StatusInternalServerError, "Failed to generate token")
        return
    }

    respondJSON(w, http.StatusOK, map[string]string{
        "access_token": accessToken,
    })
}

// Update login to return both tokens
func loginHandler(w http.ResponseWriter, r *http.Request) {
    // ... existing login logic ...

    // Generate access token (15 min)
    accessToken, _ := generateToken(user.ID, user.Username, user.Role)
    
    // Generate refresh token (7 days)
    refreshToken, _ := generateRefreshToken(user.ID)

    respondJSON(w, http.StatusOK, map[string]interface{}{
        "access_token":  accessToken,
        "refresh_token": refreshToken,
        "user":          user,
    })
}
```

### Bonus 3: Account Verification

```go
type VerificationCode struct {
    UserID    int
    Code      string
    ExpiresAt time.Time
}

var (
    verifications = make(map[int]VerificationCode)
    verifyMu      sync.RWMutex
)

// Add Verified field to User model
type User struct {
    // ... existing fields ...
    Verified bool `json:"verified"`
}

func registerHandler(w http.ResponseWriter, r *http.Request) {
    // ... existing registration logic ...

    // Generate verification code
    code := generateRandomCode(6)
    
    verifyMu.Lock()
    verifications[user.ID] = VerificationCode{
        UserID:    user.ID,
        Code:      code,
        ExpiresAt: time.Now().Add(24 * time.Hour),
    }
    verifyMu.Unlock()

    // In production: send email with code
    fmt.Printf("Verification code for %s: %s\n", user.Email, code)

    // ... return response ...
}

func verifyHandler(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Email string `json:"email"`
        Code  string `json:"code"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }

    // Find user by email
    usersMu.RLock()
    userID, exists := emails[req.Email]
    if !exists {
        usersMu.RUnlock()
        respondError(w, http.StatusNotFound, "User not found")
        return
    }
    usersMu.RUnlock()

    // Check verification code
    verifyMu.RLock()
    vc, exists := verifications[userID]
    verifyMu.RUnlock()

    if !exists || vc.Code != req.Code {
        respondError(w, http.StatusBadRequest, "Invalid verification code")
        return
    }

    if time.Now().After(vc.ExpiresAt) {
        respondError(w, http.StatusBadRequest, "Verification code expired")
        return
    }

    // Mark user as verified
    usersMu.Lock()
    user := users[userID]
    user.Verified = true
    users[userID] = user
    usersMu.Unlock()

    // Delete verification code
    verifyMu.Lock()
    delete(verifications, userID)
    verifyMu.Unlock()

    respondJSON(w, http.StatusOK, map[string]string{
        "message": "Account verified successfully",
    })
}

func generateRandomCode(length int) string {
    const charset = "0123456789"
    code := make([]byte, length)
    for i := range code {
        code[i] = charset[rand.Intn(len(charset))]
    }
    return string(code)
}
```

---

## Testing the Complete Solution

### Test Script

Create `test_auth.sh`:

```bash
#!/bin/bash

BASE="http://localhost:8080"

echo "=== Test 1: Register User ==="
REGISTER_RESPONSE=$(curl -s -X POST $BASE/register \
  -H "Content-Type: application/json" \
  -d '{
    "username":"alice",
    "email":"alice@test.com",
    "password":"SecurePass123",
    "full_name":"Alice Smith"
  }')

echo $REGISTER_RESPONSE | jq .

TOKEN=$(echo $REGISTER_RESPONSE | jq -r '.token')
echo "Token: $TOKEN"

echo -e "\n=== Test 2: Login ==="
curl -s -X POST $BASE/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"SecurePass123"}' | jq .

echo -e "\n=== Test 3: Get Profile ==="
curl -s $BASE/api/profile \
  -H "Authorization: Bearer $TOKEN" | jq .

echo -e "\n=== Test 4: Update Profile ==="
curl -s -X PUT $BASE/api/profile \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"full_name":"Alice Johnson","bio":"Go developer"}' | jq .

echo -e "\n=== Test 5: Change Password ==="
curl -s -X PUT $BASE/api/password \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"current_password":"SecurePass123","new_password":"NewSecure456"}' | jq .

echo -e "\n=== Test 6: Validation Errors ==="
echo "Weak password:"
curl -s -X POST $BASE/register \
  -H "Content-Type: application/json" \
  -d '{"username":"bob","email":"bob@test.com","password":"weak"}' | jq .

echo -e "\nDuplicate username:"
curl -s -X POST $BASE/register \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","email":"different@test.com","password":"SecurePass123"}' | jq .

echo -e "\n=== Test 7: Wrong Password ==="
curl -s -X POST $BASE/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"WrongPassword"}' | jq .

echo -e "\n=== Test 8: No Auth Token ==="
curl -s $BASE/api/profile | jq .
```

Run: `chmod +x test_auth.sh && ./test_auth.sh`

---

## What You've Learned

✅ Secure password hashing with bcrypt  
✅ JWT token generation and validation  
✅ Authentication middleware implementation  
✅ Context usage for request data  
✅ Input validation with regex  
✅ Uniqueness constraints  
✅ Protected route patterns  
✅ Security best practices  

You now have a production-ready authentication system!
