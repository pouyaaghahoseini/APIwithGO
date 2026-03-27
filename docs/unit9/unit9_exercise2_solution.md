# Unit 9 - Exercise 2 Solution: Integration Testing & CI/CD

**Complete implementation with PostgreSQL, Docker, and CI/CD**

---

## Full Solution Code

### Main Application Code

```go
// main.go
package main

import (
    "database/sql"
    "fmt"
    "log"
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
    query := `
        INSERT INTO users (email, name, password)
        VALUES ($1, $2, $3)
        RETURNING id, created_at
    `

    err := r.db.QueryRow(query, user.Email, user.Name, user.Password).
        Scan(&user.ID, &user.CreatedAt)

    if err != nil {
        return nil, err
    }

    return &user, nil
}

func (r *PostgresRepository) GetUserByEmail(email string) (*User, error) {
    query := `
        SELECT id, email, name, password, created_at
        FROM users
        WHERE email = $1
    `

    var user User
    err := r.db.QueryRow(query, email).Scan(
        &user.ID,
        &user.Email,
        &user.Name,
        &user.Password,
        &user.CreatedAt,
    )

    if err != nil {
        return nil, err
    }

    return &user, nil
}

func (r *PostgresRepository) GetUserByID(id int) (*User, error) {
    query := `
        SELECT id, email, name, password, created_at
        FROM users
        WHERE id = $1
    `

    var user User
    err := r.db.QueryRow(query, id).Scan(
        &user.ID,
        &user.Email,
        &user.Name,
        &user.Password,
        &user.CreatedAt,
    )

    if err != nil {
        return nil, err
    }

    return &user, nil
}

func (r *PostgresRepository) CreatePost(post Post) (*Post, error) {
    query := `
        INSERT INTO posts (user_id, title, content, published)
        VALUES ($1, $2, $3, $4)
        RETURNING id, created_at
    `

    err := r.db.QueryRow(query, post.UserID, post.Title, post.Content, post.Published).
        Scan(&post.ID, &post.CreatedAt)

    if err != nil {
        return nil, err
    }

    return &post, nil
}

func (r *PostgresRepository) GetPost(id int) (*Post, error) {
    query := `
        SELECT id, user_id, title, content, published, created_at
        FROM posts
        WHERE id = $1
    `

    var post Post
    err := r.db.QueryRow(query, id).Scan(
        &post.ID,
        &post.UserID,
        &post.Title,
        &post.Content,
        &post.Published,
        &post.CreatedAt,
    )

    if err != nil {
        return nil, err
    }

    return &post, nil
}

func (r *PostgresRepository) ListPosts(userID int, publishedOnly bool) ([]*Post, error) {
    var query string
    var rows *sql.Rows
    var err error

    if publishedOnly {
        query = `
            SELECT id, user_id, title, content, published, created_at
            FROM posts
            WHERE user_id = $1 AND published = true
            ORDER BY created_at DESC
        `
        rows, err = r.db.Query(query, userID)
    } else {
        query = `
            SELECT id, user_id, title, content, published, created_at
            FROM posts
            WHERE user_id = $1
            ORDER BY created_at DESC
        `
        rows, err = r.db.Query(query, userID)
    }

    if err != nil {
        return nil, err
    }
    defer rows.Close()

    posts := []*Post{}
    for rows.Next() {
        var post Post
        err := rows.Scan(
            &post.ID,
            &post.UserID,
            &post.Title,
            &post.Content,
            &post.Published,
            &post.CreatedAt,
        )
        if err != nil {
            return nil, err
        }
        posts = append(posts, &post)
    }

    return posts, nil
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

    fmt.Println("Server running on :8080")
}
```

---

## Integration Test Suite

```go
// +build integration

// integration_test.go
package main

import (
    "database/sql"
    "fmt"
    "os"
    "sync"
    "testing"
    "time"

    _ "github.com/lib/pq"
)

var testDB *sql.DB

// TestMain runs before all tests
func TestMain(m *testing.M) {
    // Get database URL from environment or use default
    dbURL := os.Getenv("DATABASE_URL")
    if dbURL == "" {
        dbURL = "postgres://postgres:test@localhost:5432/testdb?sslmode=disable"
    }

    var err error
    testDB, err = connectDB(dbURL)
    if err != nil {
        fmt.Printf("Failed to connect to test database: %v\n", err)
        os.Exit(1)
    }

    // Run migrations
    if err := runMigrations(testDB); err != nil {
        fmt.Printf("Failed to run migrations: %v\n", err)
        os.Exit(1)
    }

    // Run tests
    code := m.Run()

    // Cleanup
    testDB.Close()

    os.Exit(code)
}

// =============================================================================
// TEST HELPERS
// =============================================================================

func setupTestDB(t *testing.T) *sql.DB {
    t.Helper()
    return testDB
}

func cleanupTestDB(t *testing.T, db *sql.DB) {
    t.Helper()

    tables := []string{"posts", "users"}
    for _, table := range tables {
        _, err := db.Exec(fmt.Sprintf("TRUNCATE TABLE %s RESTART IDENTITY CASCADE", table))
        if err != nil {
            t.Fatalf("Failed to truncate table %s: %v", table, err)
        }
    }
}

func seedUsers(t *testing.T, db *sql.DB, users []User) []User {
    t.Helper()

    result := make([]User, len(users))
    for i, user := range users {
        query := `
            INSERT INTO users (email, name, password)
            VALUES ($1, $2, $3)
            RETURNING id, created_at
        `
        err := db.QueryRow(query, user.Email, user.Name, user.Password).
            Scan(&user.ID, &user.CreatedAt)

        if err != nil {
            t.Fatalf("Failed to seed user: %v", err)
        }
        result[i] = user
    }

    return result
}

func seedPosts(t *testing.T, db *sql.DB, posts []Post) []Post {
    t.Helper()

    result := make([]Post, len(posts))
    for i, post := range posts {
        query := `
            INSERT INTO posts (user_id, title, content, published)
            VALUES ($1, $2, $3, $4)
            RETURNING id, created_at
        `
        err := db.QueryRow(query, post.UserID, post.Title, post.Content, post.Published).
            Scan(&post.ID, &post.CreatedAt)

        if err != nil {
            t.Fatalf("Failed to seed post: %v", err)
        }
        result[i] = post
    }

    return result
}

// =============================================================================
// REPOSITORY TESTS
// =============================================================================

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

func TestPostgresRepository_GetUserByEmail(t *testing.T) {
    db := setupTestDB(t)
    defer cleanupTestDB(t, db)

    repo := NewPostgresRepository(db)

    // Seed user
    users := seedUsers(t, db, []User{
        {Email: "alice@example.com", Name: "Alice", Password: "password123"},
    })

    tests := []struct {
        name    string
        email   string
        wantErr bool
    }{
        {"existing user", "alice@example.com", false},
        {"non-existent user", "nobody@example.com", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            user, err := repo.GetUserByEmail(tt.email)

            if tt.wantErr {
                if err == nil {
                    t.Error("Expected error, got nil")
                }
                return
            }

            if err != nil {
                t.Fatalf("Unexpected error: %v", err)
            }

            if user.Email != tt.email {
                t.Errorf("Email: got %q, want %q", user.Email, tt.email)
            }

            if user.ID != users[0].ID {
                t.Errorf("ID: got %d, want %d", user.ID, users[0].ID)
            }
        })
    }
}

func TestPostgresRepository_CreatePost(t *testing.T) {
    db := setupTestDB(t)
    defer cleanupTestDB(t, db)

    repo := NewPostgresRepository(db)

    // Seed user
    users := seedUsers(t, db, []User{
        {Email: "alice@example.com", Name: "Alice", Password: "password123"},
    })

    post := Post{
        UserID:    users[0].ID,
        Title:     "Test Post",
        Content:   "Test content",
        Published: true,
    }

    created, err := repo.CreatePost(post)
    if err != nil {
        t.Fatalf("Failed to create post: %v", err)
    }

    if created.ID == 0 {
        t.Error("Expected post to have ID")
    }

    if created.Title != post.Title {
        t.Errorf("Title: got %q, want %q", created.Title, post.Title)
    }

    // Verify post exists in database
    var count int
    db.QueryRow("SELECT COUNT(*) FROM posts WHERE id = $1", created.ID).Scan(&count)
    if count != 1 {
        t.Errorf("Expected 1 post in DB, got %d", count)
    }
}

func TestPostgresRepository_ListPosts(t *testing.T) {
    db := setupTestDB(t)
    defer cleanupTestDB(t, db)

    repo := NewPostgresRepository(db)

    // Seed user
    users := seedUsers(t, db, []User{
        {Email: "alice@example.com", Name: "Alice", Password: "password123"},
    })

    // Seed posts
    seedPosts(t, db, []Post{
        {UserID: users[0].ID, Title: "Post 1", Published: true},
        {UserID: users[0].ID, Title: "Post 2", Published: false},
        {UserID: users[0].ID, Title: "Post 3", Published: true},
    })

    tests := []struct {
        name          string
        userID        int
        publishedOnly bool
        expectedCount int
    }{
        {"all posts", users[0].ID, false, 3},
        {"published only", users[0].ID, true, 2},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            posts, err := repo.ListPosts(tt.userID, tt.publishedOnly)
            if err != nil {
                t.Fatalf("Failed to list posts: %v", err)
            }

            if len(posts) != tt.expectedCount {
                t.Errorf("Expected %d posts, got %d", tt.expectedCount, len(posts))
            }
        })
    }
}

// =============================================================================
// WORKFLOW TESTS
// =============================================================================

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

    // Step 5: List all posts
    allPosts, err := repo.ListPosts(created.ID, false)
    if err != nil {
        t.Fatalf("Failed to list all posts: %v", err)
    }

    if len(allPosts) != 2 {
        t.Errorf("Expected 2 total posts, got %d", len(allPosts))
    }
}

func TestUserDeletionCascade(t *testing.T) {
    db := setupTestDB(t)
    defer cleanupTestDB(t, db)

    repo := NewPostgresRepository(db)

    // Create user with posts
    user, _ := repo.CreateUser(User{
        Email:    "cascade@example.com",
        Name:     "Cascade User",
        Password: "password123",
    })

    repo.CreatePost(Post{
        UserID:  user.ID,
        Title:   "Post 1",
        Content: "Content",
    })

    repo.CreatePost(Post{
        UserID:  user.ID,
        Title:   "Post 2",
        Content: "Content",
    })

    // Verify posts exist
    posts, _ := repo.ListPosts(user.ID, false)
    if len(posts) != 2 {
        t.Fatalf("Expected 2 posts, got %d", len(posts))
    }

    // Delete user
    _, err := db.Exec("DELETE FROM users WHERE id = $1", user.ID)
    if err != nil {
        t.Fatalf("Failed to delete user: %v", err)
    }

    // Verify posts were deleted (cascade)
    var postCount int
    db.QueryRow("SELECT COUNT(*) FROM posts WHERE user_id = $1", user.ID).Scan(&postCount)

    if postCount != 0 {
        t.Errorf("Expected 0 posts after user deletion, got %d", postCount)
    }
}

// =============================================================================
// TRANSACTION TESTS
// =============================================================================

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

    // Commit transaction
    tx.Commit()

    // Verify user WAS saved
    var count int
    db.QueryRow("SELECT COUNT(*) FROM users WHERE id = $1", userID).Scan(&count)

    if count != 1 {
        t.Error("User should exist after commit")
    }
}

// =============================================================================
// CONCURRENT ACCESS TESTS
// =============================================================================

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
    var wg sync.WaitGroup

    for i := 0; i < numPosts; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            _, err := repo.CreatePost(Post{
                UserID:  user.ID,
                Title:   fmt.Sprintf("Post %d", n),
                Content: "Concurrent content",
            })
            errors <- err
        }(i)
    }

    wg.Wait()
    close(errors)

    // Check for errors
    for err := range errors {
        if err != nil {
            t.Errorf("Failed to create post: %v", err)
        }
    }

    // Verify all posts were created
    posts, _ := repo.ListPosts(user.ID, false)
    if len(posts) != numPosts {
        t.Errorf("Expected %d posts, got %d", numPosts, len(posts))
    }
}

func TestConcurrentUserCreation(t *testing.T) {
    db := setupTestDB(t)
    defer cleanupTestDB(t, db)

    repo := NewPostgresRepository(db)

    numUsers := 5
    errors := make(chan error, numUsers)
    var wg sync.WaitGroup

    for i := 0; i < numUsers; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            _, err := repo.CreateUser(User{
                Email:    fmt.Sprintf("user%d@example.com", n),
                Name:     fmt.Sprintf("User %d", n),
                Password: "password123",
            })
            errors <- err
        }(i)
    }

    wg.Wait()
    close(errors)

    // Check for errors
    for err := range errors {
        if err != nil {
            t.Errorf("Failed to create user: %v", err)
        }
    }

    // Verify all users were created
    var count int
    db.QueryRow("SELECT COUNT(*) FROM users").Scan(&count)

    if count != numUsers {
        t.Errorf("Expected %d users, got %d", numUsers, count)
    }
}

// =============================================================================
// PERFORMANCE TESTS
// =============================================================================

func TestBulkInsertPerformance(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping performance test in short mode")
    }

    db := setupTestDB(t)
    defer cleanupTestDB(t, db)

    repo := NewPostgresRepository(db)

    // Create user
    user, _ := repo.CreateUser(User{
        Email:    "bulk@example.com",
        Name:     "Bulk User",
        Password: "password123",
    })

    // Create 1000 posts
    start := time.Now()
    numPosts := 1000

    for i := 0; i < numPosts; i++ {
        _, err := repo.CreatePost(Post{
            UserID:  user.ID,
            Title:   fmt.Sprintf("Post %d", i),
            Content: "Test content",
        })
        if err != nil {
            t.Fatalf("Failed to create post: %v", err)
        }
    }

    duration := time.Since(start)

    t.Logf("Created %d posts in %v", numPosts, duration)

    if duration > 10*time.Second {
        t.Errorf("Creating %d posts took too long: %v", numPosts, duration)
    }
}
```

---

## CI/CD Configuration

### GitHub Actions Workflow

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
    name: Unit Tests
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
    name: Integration Tests
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

    - name: Wait for PostgreSQL
      run: |
        until pg_isready -h localhost -p 5432 -U postgres; do
          echo "Waiting for postgres..."
          sleep 2
        done

    - name: Run integration tests
      run: go test -v -tags=integration -cover ./...
      env:
        DATABASE_URL: postgres://postgres:test@localhost:5432/testdb?sslmode=disable

    - name: Integration test coverage
      run: go test -tags=integration -coverprofile=integration-coverage.out ./...
      env:
        DATABASE_URL: postgres://postgres:test@localhost:5432/testdb?sslmode=disable

    - name: Upload integration coverage
      uses: actions/upload-artifact@v3
      with:
        name: integration-coverage
        path: integration-coverage.out
```

---

## Docker Compose Setup

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

---

## Makefile

Create `Makefile`:

```makefile
.PHONY: test test-unit test-integration test-all docker-up docker-down

test-unit:
	go test -v ./...

test-integration: docker-up
	@echo "Waiting for PostgreSQL..."
	@sleep 5
	DATABASE_URL=postgres://postgres:test@localhost:5433/testdb?sslmode=disable \
		go test -v -tags=integration ./...
	$(MAKE) docker-down

test-all: test-unit test-integration

docker-up:
	docker-compose -f docker-compose.test.yml up -d

docker-down:
	docker-compose -f docker-compose.test.yml down -v

coverage:
	go test -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out

integration-coverage: docker-up
	@sleep 5
	DATABASE_URL=postgres://postgres:test@localhost:5433/testdb?sslmode=disable \
		go test -tags=integration -coverprofile=integration-coverage.out ./...
	go tool cover -html=integration-coverage.out
	$(MAKE) docker-down

clean:
	rm -f coverage.out integration-coverage.out coverage.html
```

---

## Running Tests

### Run Unit Tests Only

```bash
make test-unit
```

### Run Integration Tests

```bash
# Start PostgreSQL and run tests
make test-integration
```

### Run All Tests

```bash
make test-all
```

### With Coverage

```bash
# Unit test coverage
make coverage

# Integration test coverage
make integration-coverage
```

---

## What You've Learned

✅ **PostgreSQL integration** with real database  
✅ **TestMain** for setup/teardown  
✅ **Build tags** to separate test types  
✅ **Database fixtures** and seeding  
✅ **Transaction testing** (commit/rollback)  
✅ **Cascade deletion** verification  
✅ **Concurrent access** testing with goroutines  
✅ **Performance testing** with benchmarks  
✅ **GitHub Actions** CI/CD pipeline  
✅ **Docker Compose** for test containers  

You now have production-grade integration testing infrastructure! 🚀
