# Unit 9 - Exercise 1: Unit Testing for User API

**Difficulty**: Intermediate  
**Estimated Time**: 45-60 minutes  
**Concepts Covered**: Unit tests, table-driven tests, HTTP handler testing, mocking, test helpers

---

## Objective

Write comprehensive unit tests for a User API that:
- Tests CRUD operations (Create, Read, Update, Delete)
- Uses table-driven tests for multiple scenarios
- Mocks the database layer
- Tests HTTP handlers with httptest
- Achieves 80%+ code coverage
- Uses test helpers for cleaner code

---

## Requirements

### API Endpoints to Test

| Method | Path | Description |
|--------|------|-------------|
| POST | /users | Create user |
| GET | /users/:id | Get user by ID |
| PUT | /users/:id | Update user |
| DELETE | /users/:id | Delete user |
| GET | /users | List all users |

### Validation Rules

- **Email**: Required, must be valid format
- **Name**: Required, min 2 characters
- **Age**: Optional, must be >= 0 if provided
- **Password**: Required for creation, min 8 characters

### Test Coverage Requirements

- ✅ Valid inputs (happy path)
- ✅ Invalid inputs (validation errors)
- ✅ Missing required fields
- ✅ Non-existent resources (404)
- ✅ Edge cases (empty strings, special characters)

---

## Starter Code

```go
package main

import (
    "encoding/json"
    "errors"
    "fmt"
    "net/http"
    "regexp"
    "strconv"

    "github.com/gorilla/mux"
)

// =============================================================================
// MODELS
// =============================================================================

type User struct {
    ID       int    `json:"id"`
    Name     string `json:"name"`
    Email    string `json:"email"`
    Age      int    `json:"age,omitempty"`
    Password string `json:"password,omitempty"`
}

type CreateUserRequest struct {
    Name     string `json:"name"`
    Email    string `json:"email"`
    Age      int    `json:"age,omitempty"`
    Password string `json:"password"`
}

type UpdateUserRequest struct {
    Name  string `json:"name,omitempty"`
    Email string `json:"email,omitempty"`
    Age   int    `json:"age,omitempty"`
}

// =============================================================================
// REPOSITORY INTERFACE
// =============================================================================

type UserRepository interface {
    Create(user User) (*User, error)
    GetByID(id int) (*User, error)
    GetByEmail(email string) (*User, error)
    Update(user User) (*User, error)
    Delete(id int) error
    List() ([]*User, error)
}

// =============================================================================
// VALIDATION
// =============================================================================

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

func validateEmail(email string) error {
    if email == "" {
        return errors.New("email is required")
    }
    if !emailRegex.MatchString(email) {
        return errors.New("invalid email format")
    }
    return nil
}

func validateName(name string) error {
    if name == "" {
        return errors.New("name is required")
    }
    if len(name) < 2 {
        return errors.New("name must be at least 2 characters")
    }
    return nil
}

func validatePassword(password string) error {
    if password == "" {
        return errors.New("password is required")
    }
    if len(password) < 8 {
        return errors.New("password must be at least 8 characters")
    }
    return nil
}

func validateAge(age int) error {
    if age < 0 {
        return errors.New("age must be non-negative")
    }
    return nil
}

// =============================================================================
// SERVICE
// =============================================================================

type UserService struct {
    repo UserRepository
}

func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) CreateUser(req CreateUserRequest) (*User, error) {
    // TODO: Implement validation
    // TODO: Check if email already exists
    // TODO: Create user
    return nil, nil
}

func (s *UserService) GetUser(id int) (*User, error) {
    // TODO: Implement
    return nil, nil
}

func (s *UserService) UpdateUser(id int, req UpdateUserRequest) (*User, error) {
    // TODO: Implement validation
    // TODO: Get existing user
    // TODO: Update fields
    // TODO: Save
    return nil, nil
}

func (s *UserService) DeleteUser(id int) error {
    // TODO: Implement
    return nil
}

func (s *UserService) ListUsers() ([]*User, error) {
    // TODO: Implement
    return nil, nil
}

// =============================================================================
// HANDLERS
// =============================================================================

type UserHandler struct {
    service *UserService
}

func NewUserHandler(service *UserService) *UserHandler {
    return &UserHandler{service: service}
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    // TODO: Implement
    // 1. Parse request body
    // 2. Call service
    // 3. Handle errors (400 for validation, 409 for conflict)
    // 4. Return 201 with user
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    // TODO: Implement
    // 1. Get ID from URL
    // 2. Call service
    // 3. Handle errors (404 if not found)
    // 4. Return 200 with user
}

func (h *UserHandler) UpdateUser(w http.ResponseWriter, r *http.Request) {
    // TODO: Implement
}

func (h *UserHandler) DeleteUser(w http.ResponseWriter, r *http.Request) {
    // TODO: Implement
}

func (h *UserHandler) ListUsers(w http.ResponseWriter, r *http.Request) {
    // TODO: Implement
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
    // In real app, use actual database
    // For testing, we'll use mock
    
    r := mux.NewRouter()
    
    fmt.Println("Server would start on :8080")
}
```

---

## Your Tasks

### Task 1: Create Mock Repository

Implement a mock repository for testing:

```go
// user_test.go
package main

import "errors"

type MockUserRepository struct {
    users   map[int]*User
    nextID  int
    emails  map[string]int // email -> user ID
}

func NewMockUserRepository() *MockUserRepository {
    return &MockUserRepository{
        users:  make(map[int]*User),
        nextID: 1,
        emails: make(map[string]int),
    }
}

// TODO: Implement Create
func (m *MockUserRepository) Create(user User) (*User, error) {
    // Check if email exists
    // Assign ID
    // Store user
    // Return user
}

// TODO: Implement GetByID
func (m *MockUserRepository) GetByID(id int) (*User, error) {
    // Return user or error if not found
}

// TODO: Implement other methods...
```

### Task 2: Write Validation Tests

Test validation functions with table-driven tests:

```go
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name    string
        email   string
        wantErr bool
    }{
        // TODO: Add test cases
        {"valid email", "test@example.com", false},
        {"empty email", "", true},
        {"missing @", "testexample.com", true},
        {"missing domain", "test@", true},
        // Add more cases
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := validateEmail(tt.email)
            if (err != nil) != tt.wantErr {
                t.Errorf("validateEmail() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}

// TODO: Write tests for validateName, validatePassword, validateAge
```

### Task 3: Write Service Tests

Test business logic:

```go
func TestUserService_CreateUser(t *testing.T) {
    tests := []struct {
        name    string
        req     CreateUserRequest
        wantErr bool
        errMsg  string
    }{
        {
            name: "valid user",
            req: CreateUserRequest{
                Name:     "Alice",
                Email:    "alice@example.com",
                Password: "secret123",
                Age:      25,
            },
            wantErr: false,
        },
        {
            name: "missing email",
            req: CreateUserRequest{
                Name:     "Bob",
                Password: "secret123",
            },
            wantErr: true,
            errMsg:  "email is required",
        },
        // TODO: Add more test cases
        // - Invalid email format
        // - Name too short
        // - Password too short
        // - Duplicate email
        // - Negative age
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            repo := NewMockUserRepository()
            service := NewUserService(repo)
            
            user, err := service.CreateUser(tt.req)
            
            if tt.wantErr {
                if err == nil {
                    t.Error("Expected error, got nil")
                }
                if tt.errMsg != "" && err.Error() != tt.errMsg {
                    t.Errorf("Expected error %q, got %q", tt.errMsg, err.Error())
                }
            } else {
                if err != nil {
                    t.Errorf("Unexpected error: %v", err)
                }
                if user.ID == 0 {
                    t.Error("Expected user to have ID")
                }
                if user.Email != tt.req.Email {
                    t.Errorf("Expected email %q, got %q", tt.req.Email, user.Email)
                }
            }
        })
    }
}

// TODO: Write tests for GetUser, UpdateUser, DeleteUser, ListUsers
```

### Task 4: Write HTTP Handler Tests

Test HTTP endpoints:

```go
func TestUserHandler_CreateUser(t *testing.T) {
    tests := []struct {
        name           string
        body           string
        expectedStatus int
        checkResponse  func(*testing.T, *httptest.ResponseRecorder)
    }{
        {
            name: "valid user",
            body: `{"name":"Alice","email":"alice@example.com","password":"secret123"}`,
            expectedStatus: http.StatusCreated,
            checkResponse: func(t *testing.T, w *httptest.ResponseRecorder) {
                var user User
                json.Unmarshal(w.Body.Bytes(), &user)
                if user.ID == 0 {
                    t.Error("Expected user to have ID")
                }
                if user.Email != "alice@example.com" {
                    t.Errorf("Expected email alice@example.com, got %s", user.Email)
                }
            },
        },
        {
            name:           "invalid json",
            body:           `{invalid}`,
            expectedStatus: http.StatusBadRequest,
        },
        {
            name:           "missing required field",
            body:           `{"name":"Alice"}`,
            expectedStatus: http.StatusBadRequest,
        },
        // TODO: Add more test cases
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            repo := NewMockUserRepository()
            service := NewUserService(repo)
            handler := NewUserHandler(service)
            
            req := httptest.NewRequest("POST", "/users", strings.NewReader(tt.body))
            w := httptest.NewRecorder()
            
            handler.CreateUser(w, req)
            
            if w.Code != tt.expectedStatus {
                t.Errorf("Expected status %d, got %d", tt.expectedStatus, w.Code)
            }
            
            if tt.checkResponse != nil {
                tt.checkResponse(t, w)
            }
        })
    }
}

// TODO: Write tests for GetUser, UpdateUser, DeleteUser, ListUsers handlers
```

### Task 5: Write Test Helpers

Create reusable test utilities:

```go
// test_helpers.go

func assertEqual(t *testing.T, got, want interface{}) {
    t.Helper()
    if got != want {
        t.Errorf("got %v, want %v", got, want)
    }
}

func assertNotNil(t *testing.T, val interface{}) {
    t.Helper()
    if val == nil {
        t.Error("expected non-nil value")
    }
}

func assertError(t *testing.T, err error, wantErr bool) {
    t.Helper()
    if (err != nil) != wantErr {
        t.Errorf("error = %v, wantErr %v", err, wantErr)
    }
}

func createTestUser(repo UserRepository, name, email string) *User {
    user := User{
        Name:     name,
        Email:    email,
        Password: "password123",
    }
    created, _ := repo.Create(user)
    return created
}
```

### Task 6: Achieve 80%+ Coverage

Run tests with coverage:

```bash
go test -cover
go test -coverprofile=coverage.out
go tool cover -html=coverage.out
```

---

## Testing Checklist

### Validation Tests
- [ ] Valid email formats
- [ ] Invalid email formats
- [ ] Empty email
- [ ] Valid names
- [ ] Names too short
- [ ] Empty names
- [ ] Valid passwords
- [ ] Passwords too short
- [ ] Valid ages
- [ ] Negative ages

### Service Tests
- [ ] Create user with valid data
- [ ] Create user with invalid data
- [ ] Create user with duplicate email
- [ ] Get existing user
- [ ] Get non-existent user
- [ ] Update user with valid data
- [ ] Update user with invalid data
- [ ] Update non-existent user
- [ ] Delete existing user
- [ ] Delete non-existent user
- [ ] List users (empty)
- [ ] List users (with data)

### Handler Tests
- [ ] POST /users with valid JSON
- [ ] POST /users with invalid JSON
- [ ] POST /users with validation errors
- [ ] GET /users/:id for existing user
- [ ] GET /users/:id for non-existent user
- [ ] GET /users/:id with invalid ID format
- [ ] PUT /users/:id with valid data
- [ ] PUT /users/:id with invalid data
- [ ] DELETE /users/:id for existing user
- [ ] DELETE /users/:id for non-existent user
- [ ] GET /users returns all users

---

## Expected Test Output

```bash
$ go test -v

=== RUN   TestValidateEmail
=== RUN   TestValidateEmail/valid_email
=== RUN   TestValidateEmail/empty_email
=== RUN   TestValidateEmail/invalid_format
--- PASS: TestValidateEmail (0.00s)
    --- PASS: TestValidateEmail/valid_email (0.00s)
    --- PASS: TestValidateEmail/empty_email (0.00s)
    --- PASS: TestValidateEmail/invalid_format (0.00s)

=== RUN   TestUserService_CreateUser
=== RUN   TestUserService_CreateUser/valid_user
=== RUN   TestUserService_CreateUser/missing_email
=== RUN   TestUserService_CreateUser/duplicate_email
--- PASS: TestUserService_CreateUser (0.00s)
    --- PASS: TestUserService_CreateUser/valid_user (0.00s)
    --- PASS: TestUserService_CreateUser/missing_email (0.00s)
    --- PASS: TestUserService_CreateUser/duplicate_email (0.00s)

=== RUN   TestUserHandler_CreateUser
=== RUN   TestUserHandler_CreateUser/valid_user
=== RUN   TestUserHandler_CreateUser/invalid_json
--- PASS: TestUserHandler_CreateUser (0.00s)
    --- PASS: TestUserHandler_CreateUser/valid_user (0.00s)
    --- PASS: TestUserHandler_CreateUser/invalid_json (0.00s)

PASS
coverage: 85.2% of statements
ok      myapp   0.123s
```

---

## Bonus Challenges

### Bonus 1: Benchmark Tests

```go
func BenchmarkValidateEmail(b *testing.B) {
    for i := 0; i < b.N; i++ {
        validateEmail("test@example.com")
    }
}

func BenchmarkUserService_CreateUser(b *testing.B) {
    repo := NewMockUserRepository()
    service := NewUserService(repo)
    
    req := CreateUserRequest{
        Name:     "Test",
        Email:    "test@example.com",
        Password: "password123",
    }
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        service.CreateUser(req)
    }
}
```

### Bonus 2: Subtests for Organization

```go
func TestUserService(t *testing.T) {
    t.Run("CreateUser", func(t *testing.T) {
        // All CreateUser tests
    })
    
    t.Run("GetUser", func(t *testing.T) {
        // All GetUser tests
    })
    
    t.Run("UpdateUser", func(t *testing.T) {
        // All UpdateUser tests
    })
}
```

### Bonus 3: Custom Assertions

```go
func assertUserEqual(t *testing.T, got, want *User) {
    t.Helper()
    if got.Name != want.Name {
        t.Errorf("name: got %q, want %q", got.Name, want.Name)
    }
    if got.Email != want.Email {
        t.Errorf("email: got %q, want %q", got.Email, want.Email)
    }
}
```

### Bonus 4: Test Fixtures

```go
var validUser = User{
    Name:     "Alice",
    Email:    "alice@example.com",
    Password: "password123",
    Age:      25,
}

func TestWithFixture(t *testing.T) {
    user := validUser // Use fixture
    // Test with user
}
```

### Bonus 5: Parallel Tests

```go
func TestConcurrentAccess(t *testing.T) {
    t.Parallel()
    
    repo := NewMockUserRepository()
    
    t.Run("CreateUser", func(t *testing.T) {
        t.Parallel()
        // Test create
    })
    
    t.Run("GetUser", func(t *testing.T) {
        t.Parallel()
        // Test get
    })
}
```

---

## What You're Learning

✅ **Table-driven tests** for comprehensive coverage  
✅ **Mock repositories** for isolated unit tests  
✅ **HTTP handler testing** with httptest  
✅ **Test helpers** for cleaner tests  
✅ **Validation testing** with edge cases  
✅ **Test organization** with subtests  
✅ **Code coverage** measurement  
✅ **Benchmark tests** for performance  

This is the foundation for test-driven development in Go!
