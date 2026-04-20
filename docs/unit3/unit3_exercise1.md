# Unit 3 - Exercise 1: Secure User Management API

**Difficulty**: Intermediate  
**Estimated Time**: 60-75 minutes  
**Concepts Covered**: User registration, login, JWT, password hashing, protected routes

---

## Objective

Build a complete user management system with secure authentication. Users can register, login, view their profile, and update their information. Implement proper password hashing and JWT-based authentication.

---

## Requirements

### Data Models

```go
type User struct {
    ID           int       `json:"id"`
    Username     string    `json:"username"`
    Email        string    `json:"email"`
    PasswordHash string    `json:"-"`  // Never expose in JSON
    FullName     string    `json:"full_name"`
    Bio          string    `json:"bio"`
    Role         string    `json:"role"`
    CreatedAt    time.Time `json:"created_at"`
    UpdatedAt    time.Time `json:"updated_at"`
}
```

### API Endpoints

| Method | Path | Auth Required | Description |
|--------|------|---------------|-------------|
| POST | /register | No | Create new user account |
| POST | /login | No | Authenticate and get token |
| GET | /api/profile | Yes | Get current user's profile |
| PUT | /api/profile | Yes | Update current user's profile |
| PUT | /api/password | Yes | Change password |
| DELETE | /api/account | Yes | Delete own account |

### Request/Response Examples

**Register (POST /register)**:
```json
Request:
{
  "username": "johndoe",
  "email": "john@example.com",
  "password": "SecurePass123",
  "full_name": "John Doe"
}

Response (201 Created):
{
  "user": {
    "id": 1,
    "username": "johndoe",
    "email": "john@example.com",
    "full_name": "John Doe",
    "bio": "",
    "role": "user",
    "created_at": "2024-01-15T10:30:00Z",
    "updated_at": "2024-01-15T10:30:00Z"
  },
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Login (POST /login)**:
```json
Request:
{
  "username": "johndoe",
  "password": "SecurePass123"
}

Response (200 OK):
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": 1,
    "username": "johndoe",
    "email": "john@example.com",
    "full_name": "John Doe",
    "role": "user"
  }
}
```

**Update Profile (PUT /api/profile)**:
```json
Request (with Authorization: Bearer <token>):
{
  "full_name": "John Smith",
  "bio": "Software developer and Go enthusiast"
}

Response (200 OK):
{
  "id": 1,
  "username": "johndoe",
  "email": "john@example.com",
  "full_name": "John Smith",
  "bio": "Software developer and Go enthusiast",
  "role": "user",
  "updated_at": "2024-01-15T11:00:00Z"
}
```

**Change Password (PUT /api/password)**:
```json
Request (with Authorization: Bearer <token>):
{
  "current_password": "SecurePass123",
  "new_password": "NewSecurePass456"
}

Response (200 OK):
{
  "message": "Password updated successfully"
}
```

### Validation Requirements

**Registration**:
- Username: 3-20 characters, alphanumeric and underscores only
- Email: Valid email format
- Password: Minimum 8 characters, must contain uppercase, lowercase, and number
- Username and email must be unique

**Login**:
- Username and password required

**Profile Update**:
- Full name: Optional, 2-50 characters if provided
- Bio: Optional, max 500 characters

**Password Change**:
- Current password must be correct
- New password must meet password requirements
- New password must be different from current password

### Security Requirements

1. **Password Hashing**: Use bcrypt with default cost
2. **JWT Tokens**: 
   - Include user_id, username, role in claims
   - 24-hour expiration
   - Sign with secret key
3. **Error Messages**: Don't reveal if username/email exists
4. **Protected Routes**: Require valid JWT token

---

## Starter Code

```go
package main

import (
    "context"
    "encoding/json"
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

var jwtSecret = []byte("change-this-secret-key")

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
// CORS MIDDLEWARE
// =============================================================================

func corsMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Allow requests from any origin (for development)
		w.Header().Set("Access-Control-Allow-Origin", "*")
		w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
		w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

		// Handle preflight OPTIONS request
		if r.Method == "OPTIONS" {
			w.WriteHeader(http.StatusOK)
			return
		}

		next.ServeHTTP(w, r)
	})
}

// TODO: Implement registerHandler
func registerHandler(w http.ResponseWriter, r *http.Request) {
    // Hints:
    // 1. Decode request body
    // 2. Validate input (use validation functions)
    // 3. Check username/email uniqueness
    // 4. Hash password with bcrypt
    // 5. Create user with default role "user"
    // 6. Generate JWT token
    // 7. Return user and token
}

// TODO: Implement loginHandler
func loginHandler(w http.ResponseWriter, r *http.Request) {
    // Hints:
    // 1. Decode request body
    // 2. Find user by username
    // 3. Verify password with bcrypt
    // 4. Generate JWT token
    // 5. Return token and user
}

// TODO: Implement getProfileHandler
func getProfileHandler(w http.ResponseWriter, r *http.Request) {
    // Hints:
    // 1. Get user from context
    // 2. Fetch full user details from storage
    // 3. Return user
}

// TODO: Implement updateProfileHandler
func updateProfileHandler(w http.ResponseWriter, r *http.Request) {
    // Hints:
    // 1. Get user from context
    // 2. Decode request body
    // 3. Validate input
    // 4. Update user fields
    // 5. Update timestamp
    // 6. Return updated user
}

// TODO: Implement changePasswordHandler
func changePasswordHandler(w http.ResponseWriter, r *http.Request) {
    // Hints:
    // 1. Get user from context
    // 2. Decode request body
    // 3. Verify current password
    // 4. Validate new password
    // 5. Check new != current
    // 6. Hash and update password
    // 7. Return success message
}

// TODO: Implement deleteAccountHandler
func deleteAccountHandler(w http.ResponseWriter, r *http.Request) {
    // Hints:
    // 1. Get user from context
    // 2. Delete user from storage
    // 3. Delete from username/email maps
    // 4. Return 204 No Content
}

// TODO: Implement validation functions
func validateUsername(username string) error {
    // Check length and format (alphanumeric + underscores)
}

func validateEmail(email string) error {
    // Check email format
}

func validatePassword(password string) error {
    // Check length and complexity
}

// TODO: Implement JWT functions
func generateToken(userID int, username, role string) (string, error) {}

func validateToken(tokenString string) (*Claims, error) {}

// TODO: Implement authMiddleware
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Extract and validate token
        // Add claims to context
    })
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
    
    // Apply CORS middleware to all routes
	  r.Use(corsMiddleware)

    // TODO: Register routes
    // Public: /register, /login
    // Protected: /api/profile (GET, PUT), /api/password (PUT), /api/account (DELETE)
    
    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", r)
}
```

---

## Testing Your API

### 1. Register a User
```bash
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "johndoe",
    "email": "john@example.com",
    "password": "SecurePass123",
    "full_name": "John Doe"
  }'
```

### 2. Login
```bash
curl -X POST http://localhost:8080/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "johndoe",
    "password": "SecurePass123"
  }'

# Save the token from response
TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### 3. Get Profile
```bash
curl http://localhost:8080/api/profile \
  -H "Authorization: Bearer $TOKEN"
```

### 4. Update Profile
```bash
curl -X PUT http://localhost:8080/api/profile \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "full_name": "John Smith",
    "bio": "Go developer"
  }'
```

### 5. Change Password
```bash
curl -X PUT http://localhost:8080/api/password \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "current_password": "SecurePass123",
    "new_password": "NewSecurePass456"
  }'
```

### 6. Delete Account
```bash
curl -X DELETE http://localhost:8080/api/account \
  -H "Authorization: Bearer $TOKEN"
```

### Test Validation
```bash
# Weak password (should fail)
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{"username":"test","email":"test@test.com","password":"weak"}'

# Duplicate username (should fail)
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{"username":"johndoe","email":"different@test.com","password":"SecurePass123"}'

# Invalid email (should fail)
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{"username":"test2","email":"notanemail","password":"SecurePass123"}'

# Wrong password (should fail)
curl -X POST http://localhost:8080/login \
  -H "Content-Type: application/json" \
  -d '{"username":"johndoe","password":"WrongPassword"}'
```

---

## Bonus Challenges

### Bonus 1: Email Login
Allow users to login with either username OR email:
```json
{
  "login": "johndoe",  // Can be username or email
  "password": "SecurePass123"
}
```

### Bonus 2: Refresh Tokens
Implement a refresh token endpoint:
- Short-lived access tokens (15 min)
- Long-lived refresh tokens (7 days)
- POST /refresh endpoint

### Bonus 3: Account Verification
Add email verification:
- Generate verification code on registration
- Require verification before login
- POST /verify endpoint

### Bonus 4: Password Reset
Implement password reset flow:
- POST /forgot-password (generate reset token)
- POST /reset-password (use token to set new password)

### Bonus 5: User Search (Admin)
Add admin-only user search:
- GET /api/admin/users?search=john
- Requires admin role

---

## Hints

### Hint 1: Password Validation
```go
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
```

### Hint 2: Email Validation
```go
func validateEmail(email string) error {
    emailRegex := regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)
    if !emailRegex.MatchString(email) {
        return errors.New("Invalid email format")
    }
    return nil
}
```

### Hint 3: Checking Uniqueness
```go
usersMu.RLock()
_, usernameExists := usernames[username]
_, emailExists := emails[email]
usersMu.RUnlock()

if usernameExists {
    return errors.New("Username already exists")
}
if emailExists {
    return errors.New("Email already exists")
}
```

### Hint 4: Password Hashing
```go
// Hash
hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
if err != nil {
    return err
}

// Verify
err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(providedPassword))
if err != nil {
    return errors.New("Invalid password")
}
```

---

## What You're Learning

✅ Secure password hashing with bcrypt  
✅ JWT token generation and validation  
✅ Authentication middleware  
✅ Protected route implementation  
✅ Input validation for security  
✅ User session management  
✅ Proper error messages (security)  
✅ Context usage for request data  

This exercise creates a production-ready authentication system!
