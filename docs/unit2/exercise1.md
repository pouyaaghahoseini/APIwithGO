# Unit 2 - Exercise 1: Task Management API

**Difficulty**: Beginner to Intermediate  
**Estimated Time**: 45-60 minutes  
**Concepts Covered**: HTTP handlers, routing, JSON, REST conventions, middleware

---

## Objective

Build a complete RESTful API for managing tasks (a TODO list). This exercise reinforces everything from Unit 2: routing, request/response handling, JSON, and middleware.

---

## Requirements

### Data Model

Create a Task struct:

```go
type Task struct {
    ID          int       `json:"id"`
    Title       string    `json:"title"`
    Description string    `json:"description"`
    Completed   bool      `json:"completed"`
    CreatedAt   time.Time `json:"created_at"`
}
```

### API Endpoints to Implement

| Method | Path | Description | Status Code |
|--------|------|-------------|-------------|
| GET | /tasks | Get all tasks | 200 |
| GET | /tasks/{id} | Get single task | 200 or 404 |
| POST | /tasks | Create new task | 201 |
| PUT | /tasks/{id} | Update task | 200 or 404 |
| DELETE | /tasks/{id} | Delete task | 204 or 404 |
| GET | /tasks?completed=true | Filter by completion status | 200 |

### Request/Response Examples

**Create Task (POST /tasks)**:
```json
Request:
{
  "title": "Learn Go",
  "description": "Complete Unit 2 exercises"
}

Response (201 Created):
{
  "id": 1,
  "title": "Learn Go",
  "description": "Complete Unit 2 exercises",
  "completed": false,
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Update Task (PUT /tasks/1)**:
```json
Request:
{
  "title": "Learn Go",
  "description": "Complete Unit 2 exercises",
  "completed": true
}

Response (200 OK):
{
  "id": 1,
  "title": "Learn Go",
  "description": "Complete Unit 2 exercises",
  "completed": true,
  "created_at": "2024-01-15T10:30:00Z"
}
```

### Validation Requirements

- Title must not be empty
- Title must be at least 3 characters
- Description is optional

Return appropriate error responses:
```json
{
  "error": "validation_failed",
  "message": "Title must be at least 3 characters"
}
```

### Middleware to Implement

1. **Logging Middleware**: Log each request (method, path, duration)
2. **CORS Middleware**: Add CORS headers for browser access
3. **Content-Type Middleware**: Set JSON content type for all API responses

---

## Starter Code

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "sync"
    "time"
    
    "github.com/gorilla/mux"
)

type Task struct {
    ID          int       `json:"id"`
    Title       string    `json:"title"`
    Description string    `json:"description"`
    Completed   bool      `json:"completed"`
    CreatedAt   time.Time `json:"created_at"`
}

type CreateTaskRequest struct {
    Title       string `json:"title"`
    Description string `json:"description"`
}

type ErrorResponse struct {
    Error   string `json:"error"`
    Message string `json:"message"`
}

// In-memory storage
var (
    tasks   = make(map[int]Task)
    nextID  = 1
    tasksMu sync.RWMutex
)

// TODO: Implement getTasks
func getTasks(w http.ResponseWriter, r *http.Request) {
    // Hint: Check for "completed" query parameter to filter
    // r.URL.Query().Get("completed")
}

// TODO: Implement getTask
func getTask(w http.ResponseWriter, r *http.Request) {
    // Hint: Use mux.Vars(r)["id"] to get the ID
}

// TODO: Implement createTask
func createTask(w http.ResponseWriter, r *http.Request) {
    // Hint: 
    // 1. Decode JSON body
    // 2. Validate input
    // 3. Create task with ID and timestamp
    // 4. Return 201 with created task
}

// TODO: Implement updateTask
func updateTask(w http.ResponseWriter, r *http.Request) {
    // Hint:
    // 1. Get ID from URL
    // 2. Check if task exists
    // 3. Decode JSON body
    // 4. Update task
    // 5. Return 200 with updated task
}

// TODO: Implement deleteTask
func deleteTask(w http.ResponseWriter, r *http.Request) {
    // Hint:
    // 1. Get ID from URL
    // 2. Check if task exists
    // 3. Delete from map
    // 4. Return 204 No Content
}

// TODO: Implement loggingMiddleware
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Log request and measure time
    })
}

// TODO: Implement corsMiddleware
func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Add CORS headers
    })
}

// TODO: Implement jsonMiddleware
func jsonMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Set Content-Type header
    })
}

// Helper functions
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, ErrorResponse{
        Error:   "error",
        Message: message,
    })
}

func main() {
    r := mux.NewRouter()
    
    // TODO: Register routes
    // TODO: Apply middleware
    
    fmt.Println("Server starting on http://localhost:8080")
    http.ListenAndServe(":8080", r)
}
```

---

## Testing Your API

Use these curl commands to test your implementation:

### Create Tasks
```bash
# Create task 1
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Learn Go","description":"Complete Unit 2"}'

# Create task 2
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Build API","description":"Create a REST API"}'

# Create task 3
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Write tests","description":"Test the API"}'
```

### Get Tasks
```bash
# Get all tasks
curl http://localhost:8080/tasks

# Get specific task
curl http://localhost:8080/tasks/1

# Filter by completed tasks
curl http://localhost:8080/tasks?completed=true

# Filter by incomplete tasks
curl http://localhost:8080/tasks?completed=false
```

### Update Task
```bash
# Mark task as completed
curl -X PUT http://localhost:8080/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"title":"Learn Go","description":"Complete Unit 2","completed":true}'
```

### Delete Task
```bash
curl -X DELETE http://localhost:8080/tasks/2
```

### Test Validation
```bash
# Should fail - title too short
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Go","description":"Short title"}'

# Should fail - empty title
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"","description":"No title"}'
```

### Test Error Handling
```bash
# Should return 404
curl http://localhost:8080/tasks/999

# Should return 404
curl -X PUT http://localhost:8080/tasks/999 \
  -H "Content-Type: application/json" \
  -d '{"title":"Test","description":"Test"}'
```

---

## Expected Output Examples

### GET /tasks (200 OK)
```json
[
  {
    "id": 1,
    "title": "Learn Go",
    "description": "Complete Unit 2",
    "completed": true,
    "created_at": "2024-01-15T10:30:00Z"
  },
  {
    "id": 2,
    "title": "Build API",
    "description": "Create a REST API",
    "completed": false,
    "created_at": "2024-01-15T10:31:00Z"
  }
]
```

### GET /tasks/1 (200 OK)
```json
{
  "id": 1,
  "title": "Learn Go",
  "description": "Complete Unit 2",
  "completed": true,
  "created_at": "2024-01-15T10:30:00Z"
}
```

### GET /tasks/999 (404 Not Found)
```json
{
  "error": "error",
  "message": "Task not found"
}
```

### POST /tasks with invalid data (400 Bad Request)
```json
{
  "error": "error",
  "message": "Title must be at least 3 characters"
}
```

### Console Output (from logging middleware)
```
[2024-01-15 10:30:00] POST /tasks - 201 - 5ms
[2024-01-15 10:30:05] GET /tasks - 200 - 1ms
[2024-01-15 10:30:10] GET /tasks/1 - 200 - 1ms
[2024-01-15 10:30:15] PUT /tasks/1 - 200 - 3ms
[2024-01-15 10:30:20] DELETE /tasks/2 - 204 - 2ms
```

---

## Bonus Challenges

### Bonus 1: Statistics Endpoint
Add a GET /tasks/stats endpoint that returns:
```json
{
  "total": 5,
  "completed": 2,
  "incomplete": 3
}
```

### Bonus 2: Partial Updates (PATCH)
Implement PATCH /tasks/{id} that only updates provided fields:
```bash
# Only update completed status
curl -X PATCH http://localhost:8080/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"completed":true}'
```

### Bonus 3: Bulk Delete
Add DELETE /tasks endpoint to delete all completed tasks:
```bash
curl -X DELETE http://localhost:8080/tasks?completed=true
```

### Bonus 4: Search
Add search functionality: GET /tasks?search=go
```bash
curl "http://localhost:8080/tasks?search=learn"
```

### Bonus 5: Sorting
Add sorting: GET /tasks?sort=created_at&order=desc
```bash
curl "http://localhost:8080/tasks?sort=title&order=asc"
```

---

## Hints

### Hint 1: Filtering Tasks
```go
func getTasks(w http.ResponseWriter, r *http.Request) {
    completedParam := r.URL.Query().Get("completed")
    
    tasksMu.RLock()
    defer tasksMu.RUnlock()
    
    taskList := []Task{}
    for _, task := range tasks {
        // If no filter, include all
        if completedParam == "" {
            taskList = append(taskList, task)
            continue
        }
        
        // Filter by completed status
        isCompleted := completedParam == "true"
        if task.Completed == isCompleted {
            taskList = append(taskList, task)
        }
    }
    
    respondJSON(w, http.StatusOK, taskList)
}
```

### Hint 2: Validation
```go
func validateTask(title string) error {
    if title == "" {
        return errors.New("Title is required")
    }
    if len(title) < 3 {
        return errors.New("Title must be at least 3 characters")
    }
    return nil
}
```

### Hint 3: Logging Middleware
```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Create a response recorder to capture status code
        recorder := &statusRecorder{ResponseWriter: w, status: 200}
        
        next.ServeHTTP(recorder, r)
        
        duration := time.Since(start)
        fmt.Printf("[%s] %s %s - %d - %v\n",
            time.Now().Format("2006-01-02 15:04:05"),
            r.Method,
            r.URL.Path,
            recorder.status,
            duration)
    })
}

type statusRecorder struct {
    http.ResponseWriter
    status int
}

func (r *statusRecorder) WriteHeader(status int) {
    r.status = status
    r.ResponseWriter.WriteHeader(status)
}
```

---

## What You're Learning

✅ Building complete RESTful APIs  
✅ Proper HTTP status codes usage  
✅ Request validation and error handling  
✅ Query parameter parsing and filtering  
✅ Middleware composition  
✅ JSON request/response handling  
✅ Concurrent access with sync.RWMutex  
✅ REST conventions in practice  

This exercise prepares you for building real-world APIs with proper structure and error handling!
