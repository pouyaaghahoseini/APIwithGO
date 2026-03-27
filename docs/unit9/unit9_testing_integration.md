# Unit 9: Testing & Integration

**Duration**: 75-90 minutes  
**Prerequisites**: Units 1-8 (Complete API development knowledge)  
**Goal**: Write comprehensive tests for your API and set up CI/CD pipelines

---

## 9.1 Why Testing?

### The Problem: Untested Code

Without tests:
```
Deploy new feature → Everything breaks
Fix bug → Create 3 new bugs
Refactor code → No way to verify it still works
New developer joins → Afraid to change anything
```

**Consequences**:
- 🐛 Bugs in production
- 😰 Fear of making changes
- 🐌 Slow development
- 💸 Expensive debugging
- 😡 Unhappy users

### The Solution: Automated Testing

With tests:
```
Write test → Write code → Test passes → Deploy confidently
Refactor code → Tests still pass → Safe to deploy
Bug reported → Write failing test → Fix bug → Test passes
```

**Benefits**:
- ✅ Confidence in code changes
- 🚀 Faster development
- 📖 Tests as documentation
- 🛡️ Catch bugs early
- 🔄 Safe refactoring

---

## 9.2 Types of Tests

### Test Pyramid

```
        /\
       /  \        E2E Tests (Few)
      /____\       - Slow, expensive
     /      \      - Test entire system
    /        \     
   /  INTEG   \    Integration Tests (Some)
  /____________\   - Test components together
 /              \  - Database, API, etc.
/     UNIT       \ Unit Tests (Many)
/_________________\ - Fast, cheap
                   - Test individual functions
```

### 1. Unit Tests

**Test individual functions in isolation**

```go
// Function to test
func Add(a, b int) int {
    return a + b
}

// Unit test
func TestAdd(t *testing.T) {
    result := Add(2, 3)
    expected := 5
    
    if result != expected {
        t.Errorf("Add(2, 3) = %d; want %d", result, expected)
    }
}
```

**Characteristics**:
- Fast (milliseconds)
- No external dependencies
- Test one thing at a time
- Easy to debug failures

---

### 2. Integration Tests

**Test components working together**

```go
// Integration test - hits real database
func TestCreateUser_Integration(t *testing.T) {
    db := setupTestDatabase()
    defer cleanupTestDatabase(db)
    
    repo := NewUserRepository(db)
    
    user := User{Name: "Alice", Email: "alice@test.com"}
    err := repo.Create(user)
    
    if err != nil {
        t.Fatalf("Failed to create user: %v", err)
    }
    
    // Verify user was saved
    saved, _ := repo.GetByEmail("alice@test.com")
    if saved.Name != "Alice" {
        t.Errorf("Expected name Alice, got %s", saved.Name)
    }
}
```

**Characteristics**:
- Slower (seconds)
- Uses real dependencies (DB, Redis, etc.)
- Tests interactions
- Closer to production

---

### 3. End-to-End (E2E) Tests

**Test complete user workflows**

```go
// E2E test - full HTTP request to response
func TestUserRegistrationFlow(t *testing.T) {
    // Start test server
    server := setupTestServer()
    defer server.Close()
    
    // 1. Register user
    resp := httpPost(server.URL+"/register", User{
        Email:    "bob@test.com",
        Password: "secret123",
    })
    assertEqual(t, resp.StatusCode, 201)
    
    // 2. Login
    resp = httpPost(server.URL+"/login", Credentials{
        Email:    "bob@test.com",
        Password: "secret123",
    })
    assertEqual(t, resp.StatusCode, 200)
    
    token := extractToken(resp)
    
    // 3. Get profile (authenticated)
    resp = httpGet(server.URL+"/profile", token)
    assertEqual(t, resp.StatusCode, 200)
}
```

**Characteristics**:
- Slowest (seconds to minutes)
- Tests entire system
- Most realistic
- Hardest to debug

---

## 9.3 Go Testing Basics

### Test File Naming

```
main.go      → main_test.go
user.go      → user_test.go
handlers.go  → handlers_test.go
```

**Convention**: `*_test.go`

### Basic Test Structure

```go
package mypackage

import "testing"

func TestFunctionName(t *testing.T) {
    // Arrange - setup
    input := "test"
    expected := "TEST"
    
    // Act - call function
    result := ToUpper(input)
    
    // Assert - verify result
    if result != expected {
        t.Errorf("ToUpper(%q) = %q; want %q", input, result, expected)
    }
}
```

### Running Tests

```bash
# Run all tests
go test

# Run tests with verbose output
go test -v

# Run specific test
go test -run TestFunctionName

# Run tests with coverage
go test -cover

# Generate coverage report
go test -coverprofile=coverage.out
go tool cover -html=coverage.out
```

---

## 9.4 Table-Driven Tests

**Best practice in Go: test multiple cases efficiently**

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a        int
        b        int
        expected int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -2, -3, -5},
        {"zero", 0, 0, 0},
        {"mixed", -5, 10, 5},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Add(%d, %d) = %d; want %d", 
                    tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

**Benefits**:
- Test many cases with less code
- Easy to add new test cases
- Clear test names
- Isolated failures

---

## 9.5 Testing HTTP Handlers

### Using httptest Package

```go
func TestGetUserHandler(t *testing.T) {
    // Create request
    req := httptest.NewRequest("GET", "/users/1", nil)
    
    // Create response recorder
    w := httptest.NewRecorder()
    
    // Call handler
    getUserHandler(w, req)
    
    // Check status code
    if w.Code != http.StatusOK {
        t.Errorf("Expected status 200, got %d", w.Code)
    }
    
    // Check response body
    var user User
    json.Unmarshal(w.Body.Bytes(), &user)
    
    if user.ID != 1 {
        t.Errorf("Expected user ID 1, got %d", user.ID)
    }
}
```

### Testing with Middleware

```go
func TestProtectedEndpoint(t *testing.T) {
    // Setup router with middleware
    r := mux.NewRouter()
    r.Use(authMiddleware)
    r.HandleFunc("/protected", protectedHandler)
    
    // Test without auth
    req := httptest.NewRequest("GET", "/protected", nil)
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)
    
    if w.Code != http.StatusUnauthorized {
        t.Errorf("Expected 401, got %d", w.Code)
    }
    
    // Test with auth
    req = httptest.NewRequest("GET", "/protected", nil)
    req.Header.Set("Authorization", "Bearer valid-token")
    w = httptest.NewRecorder()
    r.ServeHTTP(w, req)
    
    if w.Code != http.StatusOK {
        t.Errorf("Expected 200, got %d", w.Code)
    }
}
```

---

## 9.6 Mocking and Test Doubles

### Interface-Based Mocking

```go
// Interface
type UserRepository interface {
    GetByID(id int) (*User, error)
    Create(user User) error
}

// Real implementation
type PostgresUserRepository struct {
    db *sql.DB
}

func (r *PostgresUserRepository) GetByID(id int) (*User, error) {
    // Query database
}

// Mock implementation for testing
type MockUserRepository struct {
    users map[int]*User
}

func (m *MockUserRepository) GetByID(id int) (*User, error) {
    user, exists := m.users[id]
    if !exists {
        return nil, errors.New("user not found")
    }
    return user, nil
}

// Test using mock
func TestUserService(t *testing.T) {
    // Use mock instead of real database
    mock := &MockUserRepository{
        users: map[int]*User{
            1: {ID: 1, Name: "Alice"},
        },
    }
    
    service := NewUserService(mock)
    
    user, err := service.GetUser(1)
    if err != nil {
        t.Fatalf("Unexpected error: %v", err)
    }
    
    if user.Name != "Alice" {
        t.Errorf("Expected Alice, got %s", user.Name)
    }
}
```

---

## 9.7 Test Database Setup

### Option 1: In-Memory SQLite

```go
func setupTestDB() *sql.DB {
    db, err := sql.Open("sqlite3", ":memory:")
    if err != nil {
        panic(err)
    }
    
    // Run migrations
    _, err = db.Exec(`
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            name TEXT,
            email TEXT UNIQUE
        )
    `)
    
    if err != nil {
        panic(err)
    }
    
    return db
}

func TestWithDatabase(t *testing.T) {
    db := setupTestDB()
    defer db.Close()
    
    // Test database operations
}
```

### Option 2: Docker Test Container

```go
func setupPostgresContainer(t *testing.T) *sql.DB {
    // Start PostgreSQL container
    cmd := exec.Command("docker", "run", "-d",
        "-p", "5433:5432",
        "-e", "POSTGRES_PASSWORD=test",
        "postgres:14")
    
    cmd.Run()
    
    // Wait for container to be ready
    time.Sleep(5 * time.Second)
    
    db, err := sql.Open("postgres", 
        "postgres://postgres:test@localhost:5433/postgres?sslmode=disable")
    
    if err != nil {
        t.Fatalf("Failed to connect: %v", err)
    }
    
    return db
}
```

### Option 3: Test Fixtures

```go
func seedTestData(db *sql.DB) {
    users := []User{
        {ID: 1, Name: "Alice", Email: "alice@test.com"},
        {ID: 2, Name: "Bob", Email: "bob@test.com"},
    }
    
    for _, user := range users {
        db.Exec("INSERT INTO users (id, name, email) VALUES (?, ?, ?)",
            user.ID, user.Name, user.Email)
    }
}

func TestWithFixtures(t *testing.T) {
    db := setupTestDB()
    defer db.Close()
    
    seedTestData(db)
    
    // Tests can assume data exists
}
```

---

## 9.8 Testing Best Practices

### ✅ DO

1. **Write tests first (TDD)**
   ```go
   // 1. Write failing test
   func TestAdd(t *testing.T) {
       result := Add(2, 3)
       if result != 5 {
           t.Error("Failed")
       }
   }
   
   // 2. Write minimal code to pass
   func Add(a, b int) int {
       return a + b
   }
   
   // 3. Refactor
   ```

2. **Use table-driven tests**
3. **Test edge cases**
   - Empty inputs
   - Null values
   - Maximum values
   - Negative numbers

4. **Use meaningful test names**
   ```go
   // Good
   func TestGetUser_WhenUserExists_ReturnsUser(t *testing.T)
   func TestGetUser_WhenUserNotFound_ReturnsError(t *testing.T)
   
   // Bad
   func TestGetUser1(t *testing.T)
   func TestGetUser2(t *testing.T)
   ```

5. **Keep tests independent**
   - Each test should set up its own data
   - Tests should not depend on each other

6. **Use test helpers**
   ```go
   func assertEqual(t *testing.T, got, want interface{}) {
       t.Helper()
       if got != want {
           t.Errorf("got %v, want %v", got, want)
       }
   }
   ```

### ❌ DON'T

1. **Don't test external libraries**
2. **Don't write tests that depend on network**
3. **Don't use sleeps** (use channels/synchronization)
4. **Don't test implementation details**
5. **Don't skip cleanup**

---

## 9.9 Benchmarking

### Basic Benchmark

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(2, 3)
    }
}

// Run benchmarks
// go test -bench=.
```

### Comparing Performance

```go
func BenchmarkCacheLookup(b *testing.B) {
    cache := setupCache()
    
    b.ResetTimer() // Don't count setup time
    
    for i := 0; i < b.N; i++ {
        cache.Get("key")
    }
}

func BenchmarkDatabaseLookup(b *testing.B) {
    db := setupDB()
    
    b.ResetTimer()
    
    for i := 0; i < b.N; i++ {
        db.Query("SELECT * FROM users WHERE id = 1")
    }
}
```

---

## 9.10 Code Coverage

### Measuring Coverage

```bash
# Run tests with coverage
go test -cover

# Generate coverage report
go test -coverprofile=coverage.out

# View HTML coverage report
go tool cover -html=coverage.out

# Coverage by function
go tool cover -func=coverage.out
```

### Coverage Goals

- **Aim for 70-80%** overall coverage
- **Critical code** (auth, payments) should be 90%+
- **Don't chase 100%** - diminishing returns
- **Focus on quality** over quantity

---

## 9.11 CI/CD Integration

### GitHub Actions Example

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up Go
      uses: actions/setup-go@v2
      with:
        go-version: 1.21
    
    - name: Run tests
      run: go test -v -cover ./...
    
    - name: Run integration tests
      run: go test -v -tags=integration ./...
      env:
        DATABASE_URL: postgres://postgres:test@localhost:5432/postgres
```

### Build Tags for Integration Tests

```go
// +build integration

package mypackage

func TestIntegration(t *testing.T) {
    // Only runs with: go test -tags=integration
}
```

---

## 9.12 Complete Testing Example

```go
package main

import (
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"
)

// Handler to test
func createUserHandler(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    
    if user.Email == "" {
        http.Error(w, "Email required", http.StatusBadRequest)
        return
    }
    
    // Save user (mocked in test)
    user.ID = 1
    
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}

// Table-driven test
func TestCreateUser(t *testing.T) {
    tests := []struct {
        name           string
        body           string
        expectedStatus int
        expectedError  string
    }{
        {
            name:           "valid user",
            body:           `{"email":"test@example.com","name":"Test"}`,
            expectedStatus: http.StatusCreated,
        },
        {
            name:           "missing email",
            body:           `{"name":"Test"}`,
            expectedStatus: http.StatusBadRequest,
            expectedError:  "Email required",
        },
        {
            name:           "invalid json",
            body:           `{invalid}`,
            expectedStatus: http.StatusBadRequest,
            expectedError:  "Invalid JSON",
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            req := httptest.NewRequest("POST", "/users", 
                strings.NewReader(tt.body))
            w := httptest.NewRecorder()
            
            createUserHandler(w, req)
            
            if w.Code != tt.expectedStatus {
                t.Errorf("Expected status %d, got %d", 
                    tt.expectedStatus, w.Code)
            }
            
            if tt.expectedError != "" {
                body := strings.TrimSpace(w.Body.String())
                if !strings.Contains(body, tt.expectedError) {
                    t.Errorf("Expected error containing %q, got %q",
                        tt.expectedError, body)
                }
            }
        })
    }
}
```

---

## Key Takeaways

✅ **Unit tests** are fast and test individual functions  
✅ **Integration tests** verify components work together  
✅ **Table-driven tests** reduce boilerplate  
✅ **httptest** package for testing HTTP handlers  
✅ **Mocking** isolates code under test  
✅ **Test coverage** should be 70-80%  
✅ **CI/CD** runs tests automatically  
✅ **Good tests** are fast, independent, and repeatable  

---

## What's Next?

Congratulations! You've completed the Go API Development course! 🎉

You now know:
- Go fundamentals
- HTTP servers
- Authentication & Authorization
- API Versioning
- Documentation
- Caching
- Pagination
- Rate Limiting
- **Testing & Integration** ← You are here

You're ready to build production-grade APIs! 🚀
