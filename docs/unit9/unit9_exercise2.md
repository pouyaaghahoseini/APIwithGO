# Unit 9 - Exercise 2: Integration Testing & CI/CD

**Difficulty**: Advanced  
**Estimated Time**: 60-75 minutes  
**Concepts Covered**: Integration tests, Docker test containers, database testing, test cleanup, CI/CD pipelines

---

## Objective

Set up integration testing infrastructure that:
- Tests with a real PostgreSQL database
- Uses Docker containers for test isolation
- Implements proper setup and teardown
- Tests complete API workflows (E2E)
- Configures GitHub Actions CI/CD
- Separates unit and integration tests with build tags

---

## Requirements

### Integration Test Scenarios

1. **User Registration Flow**
   - Register user → Login → Get profile
   - Verify data persists in database

2. **Post Creation and Retrieval**
   - Create post → Get post → List posts
   - Test pagination and filtering

3. **Concurrent Access**
   - Multiple users creating posts simultaneously
   - Verify no race conditions or data corruption

4. **Transaction Rollback**
   - Start transaction → Create user → Rollback
   - Verify user not saved

### Test Infrastructure

- **Database**: PostgreSQL in Docker
- **Migrations**: Automatic schema setup
- **Cleanup**: Truncate tables between tests
- **Fixtures**: Seed test data
- **Build Tags**: Separate unit and integration tests

---

## Starter Code

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "net/http"
    "time"

    _ "github.com/lib/pq"
)

// =============================================================================
// MODELS
// =============================================================================

type User struct {
    ID        int       `json:"id"`
    Email     string    `json:"email"`
    Name      string    `json:"name"`
    Password  string    `json:"-"`
    CreatedAt time.Time `json:"created_at"`
}

type Post struct {
    ID        int       `json:"id"`
    UserID    int       `json:"user_id"`
    Title     string    `json:"title"`
    Content   string    `json:"content"`
    Published bool      `json:"published"`
    CreatedAt time.Time `json:"created_at"`
}

// =============================================================================
// REPOSITORY
// =============================================================================

type PostgresRepository struct {
    db *sql.DB
}

func NewPostgresRepository(db *sql.DB) *PostgresRepository {
    return &PostgresRepository{db: db}
}

func (r *PostgresRepository) CreateUser(user User) (*User, error) {
    // TODO: Implement
    // INSERT INTO users ... RETURNING id, created_at
    return nil, nil
}

func (r *PostgresRepository) GetUserByEmail(email string) (*User, error) {
    // TODO: Implement
    return nil, nil
}

func (r *PostgresRepository) CreatePost(post Post) (*Post, error) {
    // TODO: Implement
    return nil, nil
}

func (r *PostgresRepository) GetPost(id int) (*Post, error) {
    // TODO: Implement
    return nil, nil
}

func (r *PostgresRepository) ListPosts(userID int, published bool) ([]*Post, error) {
    // TODO: Implement
    return nil, nil
}

// =============================================================================
// DATABASE SETUP
// =============================================================================

func connectDB(connStr string) (*sql.DB, error) {
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        return nil, err
    }

    // Test connection
    if err := db.Ping(); err != nil {
        return nil, err
    }

    return db, nil
}

func runMigrations(db *sql.DB) error {
    migrations := `
    CREATE TABLE IF NOT EXISTS users (
        id SERIAL PRIMARY KEY,
        email VARCHAR(255) UNIQUE NOT NULL,
        name VARCHAR(255) NOT NULL,
        password VARCHAR(255) NOT NULL,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );

    CREATE TABLE IF NOT EXISTS posts (
        id SERIAL PRIMARY KEY,
        user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
        title VARCHAR(255) NOT NULL,
        content TEXT,
        published BOOLEAN DEFAULT FALSE,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );

    CREATE INDEX IF NOT EXISTS idx_posts_user_id ON posts(user_id);
    CREATE INDEX IF NOT EXISTS idx_posts_published ON posts(published);
    `

    _, err := db.Exec(migrations)
    return err
}

// =============================================================================
// MAIN
// =============================================================================

func main() {
    connStr := "postgres://postgres:password@localhost:5432/myapp?sslmode=disable"
    
    db, err := connectDB(connStr)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    if err := runMigrations(db); err != nil {
        log.Fatal(err)
    }

    fmt.Println("Server would start on :8080")
}
```

---

## Your Tasks

### Task 1: Create Test Database Setup

Create integration test infrastructure:

```go
// +build integration

package main

import (
    "database/sql"
    "fmt"
    "os"
    "testing"
    "time"
)

var testDB *sql.DB

func TestMain(m *testing.M) {
    // TODO: Setup - runs before all tests
    // 1. Start Docker container or connect to test DB
    // 2. Run migrations
    // 3. Run tests
    // 4. Cleanup - stop container

    code := m.Run()
    
    // Cleanup
    if testDB != nil {
        testDB.Close()
    }
    
    os.Exit(code)
}

func setupTestDB(t *testing.T) *sql.DB {
    // TODO: Connect to test database
    // Return connection
}

func cleanupTestDB(t *testing.T, db *sql.DB) {
    // TODO: Truncate all tables
    // Reset sequences
    t.Helper()
    
    tables := []string{"posts", "users"}
    for _, table := range tables {
        db.Exec(fmt.Sprintf("TRUNCATE TABLE %s CASCADE", table))
    }
}

func seedUsers(t *testing.T, db *sql.DB, users []User) []User {
    // TODO: Insert test users
    // Return users with IDs
    t.Helper()
}

func seedPosts(t *testing.T, db *sql.DB, posts []Post) []Post {
    // TODO: Insert test posts
    t.Helper()
}
```

### Task 2: Write Repository Integration Tests

Test database operations:

```go
// +build integration

func TestPostgresRepository_CreateUser(t *testing.T) {
    db := setupTestDB(t)
    defer cleanupTestDB(t, db)
    
    repo := NewPostgresRepository(db)
    
    tests := []struct {
        name    string
        user    User
        wantErr bool
    }{
        {
            name: "valid user",
            user: User{
                Email:    "test@example.com",
                Name:     "Test User",
                Password: "password123",
            },
            wantErr: false,
        },
        {
            name: "duplicate email",
            user: User{
                Email:    "test@example.com",
                Name:     "Another User",
                Password: "password123",
            },
            wantErr: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            created, err := repo.CreateUser(tt.user)
            
            if tt.wantErr {
                if err == nil {
                    t.Error("Expected error, got nil")
                }
                return
            }
            
            if err != nil {
                t.Fatalf("Unexpected error: %v", err)
            }
            
            // Verify user was created
            if created.ID == 0 {
                t.Error("Expected user to have ID")
            }
            
            if created.Email != tt.user.Email {
                t.Errorf("Email: got %q, want %q", created.Email, tt.user.Email)
            }
            
            // Verify user exists in database
            var count int
            db.QueryRow("SELECT COUNT(*) FROM users WHERE id = $1", created.ID).Scan(&count)
            if count != 1 {
                t.Errorf("Expected 1 user in DB, got %d", count)
            }
        })
    }
}

// TODO: Write tests for:
// - GetUserByEmail
// - CreatePost
// - GetPost
// - ListPosts (with filters)
```

### Task 3: Write End-to-End Workflow Tests

Test complete user workflows:

```go
// +build integration

func TestUserRegistrationWorkflow(t *testing.T) {
    db := setupTestDB(t)
    defer cleanupTestDB(t, db)
    
    repo := NewPostgresRepository(db)
    
    // Step 1: Create user
    user := User{
        Email:    "alice@example.com",
        Name:     "Alice",
        Password: "password123",
    }
    
    created, err := repo.CreateUser(user)
    if err != nil {
        t.Fatalf("Failed to create user: %v", err)
    }
    
    // Step 2: Verify user can be retrieved by email
    retrieved, err := repo.GetUserByEmail("alice@example.com")
    if err != nil {
        t.Fatalf("Failed to get user: %v", err)
    }
    
    if retrieved.ID != created.ID {
        t.Errorf("User ID mismatch: got %d, want %d", retrieved.ID, created.ID)
    }
    
    // Step 3: Create posts for user
    post1, _ := repo.CreatePost(Post{
        UserID:    created.ID,
        Title:     "First Post",
        Content:   "Hello World",
        Published: true,
    })
    
    post2, _ := repo.CreatePost(Post{
        UserID:    created.ID,
        Title:     "Draft Post",
        Content:   "Work in progress",
        Published: false,
    })
    
    // Step 4: List published posts
    posts, err := repo.ListPosts(created.ID, true)
    if err != nil {
        t.Fatalf("Failed to list posts: %v", err)
    }
    
    if len(posts) != 1 {
        t.Errorf("Expected 1 published post, got %d", len(posts))
    }
    
    if posts[0].ID != post1.ID {
        t.Error("Expected to get post1")
    }
}

// TODO: Write workflow tests for:
// - Creating multiple posts and filtering
// - User with posts being deleted (cascade)
// - Concurrent post creation
```

### Task 4: Test Transaction Handling

```go
// +build integration

func TestTransactionRollback(t *testing.T) {
    db := setupTestDB(t)
    defer cleanupTestDB(t, db)
    
    // Start transaction
    tx, err := db.Begin()
    if err != nil {
        t.Fatal(err)
    }
    
    // Create user in transaction
    var userID int
    err = tx.QueryRow(`
        INSERT INTO users (email, name, password)
        VALUES ($1, $2, $3)
        RETURNING id
    `, "test@example.com", "Test", "password").Scan(&userID)
    
    if err != nil {
        t.Fatal(err)
    }
    
    // Rollback transaction
    tx.Rollback()
    
    // Verify user was NOT saved
    var count int
    db.QueryRow("SELECT COUNT(*) FROM users WHERE id = $1", userID).Scan(&count)
    
    if count != 0 {
        t.Error("User should not exist after rollback")
    }
}

func TestTransactionCommit(t *testing.T) {
    // TODO: Test successful transaction commit
}
```

### Task 5: Test Concurrent Access

```go
// +build integration

func TestConcurrentPostCreation(t *testing.T) {
    db := setupTestDB(t)
    defer cleanupTestDB(t, db)
    
    repo := NewPostgresRepository(db)
    
    // Create test user
    user, _ := repo.CreateUser(User{
        Email:    "concurrent@example.com",
        Name:     "Concurrent User",
        Password: "password123",
    })
    
    // Create 10 posts concurrently
    numPosts := 10
    errors := make(chan error, numPosts)
    
    for i := 0; i < numPosts; i++ {
        go func(n int) {
            _, err := repo.CreatePost(Post{
                UserID:  user.ID,
                Title:   fmt.Sprintf("Post %d", n),
                Content: "Concurrent content",
            })
            errors <- err
        }(i)
    }
    
    // Wait for all goroutines
    for i := 0; i < numPosts; i++ {
        if err := <-errors; err != nil {
            t.Errorf("Failed to create post: %v", err)
        }
    }
    
    // Verify all posts were created
    posts, _ := repo.ListPosts(user.ID, false)
    if len(posts) != numPosts {
        t.Errorf("Expected %d posts, got %d", numPosts, len(posts))
    }
}
```

### Task 6: Setup GitHub Actions CI/CD

Create `.github/workflows/test.yml`:

```yaml
name: Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Go
      uses: actions/setup-go@v3
      with:
        go-version: '1.21'
    
    - name: Run unit tests
      run: go test -v -cover ./...
    
    - name: Generate coverage report
      run: |
        go test -coverprofile=coverage.out ./...
        go tool cover -html=coverage.out -o coverage.html
    
    - name: Upload coverage
      uses: actions/upload-artifact@v3
      with:
        name: coverage-report
        path: coverage.html

  integration-tests:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Go
      uses: actions/setup-go@v3
      with:
        go-version: '1.21'
    
    - name: Run integration tests
      run: go test -v -tags=integration ./...
      env:
        DATABASE_URL: postgres://postgres:test@localhost:5432/testdb?sslmode=disable
    
    - name: Check integration test coverage
      run: go test -tags=integration -cover ./...
```

---

## Running Tests

### Run Unit Tests Only

```bash
go test -v ./...
```

### Run Integration Tests Only

```bash
go test -v -tags=integration ./...
```

### Run All Tests

```bash
go test -v -tags=integration ./...
```

### With Coverage

```bash
# Unit tests coverage
go test -cover ./...

# Integration tests coverage
go test -tags=integration -cover ./...
```

---

## Docker Test Container Setup

### Option 1: Docker Compose

Create `docker-compose.test.yml`:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:14
    environment:
      POSTGRES_PASSWORD: test
      POSTGRES_DB: testdb
    ports:
      - "5433:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

Run tests:
```bash
docker-compose -f docker-compose.test.yml up -d
go test -tags=integration ./...
docker-compose -f docker-compose.test.yml down
```

### Option 2: Makefile

Create `Makefile`:

```makefile
.PHONY: test test-unit test-integration test-all

test-unit:
	go test -v ./...

test-integration:
	docker-compose -f docker-compose.test.yml up -d
	sleep 5
	go test -v -tags=integration ./...
	docker-compose -f docker-compose.test.yml down

test-all: test-unit test-integration

coverage:
	go test -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out
```

---

## Expected Test Output

```bash
$ make test-integration

Starting postgres container...
Running integration tests...

=== RUN   TestPostgresRepository_CreateUser
=== RUN   TestPostgresRepository_CreateUser/valid_user
=== RUN   TestPostgresRepository_CreateUser/duplicate_email
--- PASS: TestPostgresRepository_CreateUser (0.05s)
    --- PASS: TestPostgresRepository_CreateUser/valid_user (0.02s)
    --- PASS: TestPostgresRepository_CreateUser/duplicate_email (0.03s)

=== RUN   TestUserRegistrationWorkflow
--- PASS: TestUserRegistrationWorkflow (0.08s)

=== RUN   TestConcurrentPostCreation
--- PASS: TestConcurrentPostCreation (0.12s)

PASS
coverage: 78.5% of statements
ok      myapp   0.342s

Stopping postgres container...
```

---

## Bonus Challenges

### Bonus 1: Test Helpers

```go
func mustCreateUser(t *testing.T, repo *PostgresRepository, email string) *User {
    t.Helper()
    user, err := repo.CreateUser(User{
        Email:    email,
        Name:     "Test User",
        Password: "password123",
    })
    if err != nil {
        t.Fatalf("Failed to create user: %v", err)
    }
    return user
}
```

### Bonus 2: Fixtures from Files

```go
func loadFixtures(t *testing.T, db *sql.DB, filename string) {
    t.Helper()
    data, _ := os.ReadFile(filename)
    _, err := db.Exec(string(data))
    if err != nil {
        t.Fatalf("Failed to load fixtures: %v", err)
    }
}
```

### Bonus 3: Database Snapshots

```go
func snapshotDB(t *testing.T, db *sql.DB) func() {
    // Create snapshot
    db.Exec("BEGIN")
    
    // Return restore function
    return func() {
        db.Exec("ROLLBACK")
    }
}
```

### Bonus 4: Performance Tests

```go
func TestPostCreationPerformance(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping performance test in short mode")
    }
    
    // Test creating 1000 posts
    start := time.Now()
    
    for i := 0; i < 1000; i++ {
        repo.CreatePost(...)
    }
    
    duration := time.Since(start)
    
    if duration > 5*time.Second {
        t.Errorf("Creating 1000 posts took too long: %v", duration)
    }
}
```

### Bonus 5: Cleanup Validation

```go
func TestCleanupRemovesAllData(t *testing.T) {
    db := setupTestDB(t)
    
    // Create test data
    repo.CreateUser(...)
    repo.CreatePost(...)
    
    // Run cleanup
    cleanupTestDB(t, db)
    
    // Verify all tables empty
    var userCount, postCount int
    db.QueryRow("SELECT COUNT(*) FROM users").Scan(&userCount)
    db.QueryRow("SELECT COUNT(*) FROM posts").Scan(&postCount)
    
    if userCount != 0 || postCount != 0 {
        t.Error("Cleanup did not remove all data")
    }
}
```

---

## What You're Learning

✅ **Integration testing** with real databases  
✅ **Docker containers** for test isolation  
✅ **Build tags** to separate test types  
✅ **Test setup/teardown** with TestMain  
✅ **Database fixtures** and seeding  
✅ **Transaction testing** (commit/rollback)  
✅ **Concurrent access** testing  
✅ **CI/CD pipelines** with GitHub Actions  
✅ **End-to-end workflows**  

You now have a complete testing infrastructure for production APIs! 🚀
