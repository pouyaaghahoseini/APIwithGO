# Unit 2 - Exercise 1 Solution: Task Management API

**Complete implementation with explanations**

---

## Full Solution Code

```go
package main

import (
    "encoding/json"
    "errors"
    "fmt"
    "net/http"
    "strconv"
    "strings"
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

type UpdateTaskRequest struct {
    Title       string `json:"title"`
    Description string `json:"description"`
    Completed   bool   `json:"completed"`
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

// getTasks returns all tasks, optionally filtered by completion status
func getTasks(w http.ResponseWriter, r *http.Request) {
    completedParam := r.URL.Query().Get("completed")

    tasksMu.RLock()
    defer tasksMu.RUnlock()

    taskList := []Task{}
    for _, task := range tasks {
        // No filter - include all tasks
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

// getTask returns a single task by ID
func getTask(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        respondError(w, http.StatusBadRequest, "validation_failed", "Invalid task ID")
        return
    }

    tasksMu.RLock()
    task, exists := tasks[id]
    tasksMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "not_found", "Task not found")
        return
    }

    respondJSON(w, http.StatusOK, task)
}

// createTask creates a new task
func createTask(w http.ResponseWriter, r *http.Request) {
    var req CreateTaskRequest
    err := json.NewDecoder(r.Body).Decode(&req)
    if err != nil {
        respondError(w, http.StatusBadRequest, "invalid_json", "Invalid JSON format")
        return
    }
    defer r.Body.Close()

    // Validate input
    if err := validateTaskInput(req.Title); err != nil {
        respondError(w, http.StatusBadRequest, "validation_failed", err.Error())
        return
    }

    // Create task
    tasksMu.Lock()
    task := Task{
        ID:          nextID,
        Title:       req.Title,
        Description: req.Description,
        Completed:   false,
        CreatedAt:   time.Now(),
    }
    tasks[nextID] = task
    nextID++
    tasksMu.Unlock()

    respondJSON(w, http.StatusCreated, task)
}

// updateTask updates an existing task
func updateTask(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        respondError(w, http.StatusBadRequest, "validation_failed", "Invalid task ID")
        return
    }

    var req UpdateTaskRequest
    err = json.NewDecoder(r.Body).Decode(&req)
    if err != nil {
        respondError(w, http.StatusBadRequest, "invalid_json", "Invalid JSON format")
        return
    }
    defer r.Body.Close()

    // Validate input
    if err := validateTaskInput(req.Title); err != nil {
        respondError(w, http.StatusBadRequest, "validation_failed", err.Error())
        return
    }

    // Check if task exists and update
    tasksMu.Lock()
    task, exists := tasks[id]
    if !exists {
        tasksMu.Unlock()
        respondError(w, http.StatusNotFound, "not_found", "Task not found")
        return
    }

    // Update task fields
    task.Title = req.Title
    task.Description = req.Description
    task.Completed = req.Completed
    tasks[id] = task
    tasksMu.Unlock()

    respondJSON(w, http.StatusOK, task)
}

// deleteTask deletes a task by ID
func deleteTask(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        respondError(w, http.StatusBadRequest, "validation_failed", "Invalid task ID")
        return
    }

    tasksMu.Lock()
    _, exists := tasks[id]
    if !exists {
        tasksMu.Unlock()
        respondError(w, http.StatusNotFound, "not_found", "Task not found")
        return
    }

    delete(tasks, id)
    tasksMu.Unlock()

    w.WriteHeader(http.StatusNoContent)
}

// validateTaskInput validates task title
func validateTaskInput(title string) error {
    title = strings.TrimSpace(title)
    if title == "" {
        return errors.New("Title is required")
    }
    if len(title) < 3 {
        return errors.New("Title must be at least 3 characters")
    }
    return nil
}

// loggingMiddleware logs each request with timing
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // Create a custom response writer to capture status code
        recorder := &statusRecorder{
            ResponseWriter: w,
            status:         http.StatusOK,
        }

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

// statusRecorder wraps ResponseWriter to capture status code
type statusRecorder struct {
    http.ResponseWriter
    status int
}

func (r *statusRecorder) WriteHeader(status int) {
    r.status = status
    r.ResponseWriter.WriteHeader(status)
}

// corsMiddleware adds CORS headers
func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        // Handle preflight requests
        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }

        next.ServeHTTP(w, r)
    })
}

// jsonMiddleware sets JSON content type for all responses
func jsonMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        next.ServeHTTP(w, r)
    })
}

// Helper functions
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, err, message string) {
    respondJSON(w, status, ErrorResponse{
        Error:   err,
        Message: message,
    })
}

func main() {
    r := mux.NewRouter()

    // Register routes
    r.HandleFunc("/tasks", getTasks).Methods("GET")
    r.HandleFunc("/tasks/{id}", getTask).Methods("GET")
    r.HandleFunc("/tasks", createTask).Methods("POST")
    r.HandleFunc("/tasks/{id}", updateTask).Methods("PUT")
    r.HandleFunc("/tasks/{id}", deleteTask).Methods("DELETE")

    // Apply middleware (order matters!)
    r.Use(loggingMiddleware)
    r.Use(corsMiddleware)
    r.Use(jsonMiddleware)

    fmt.Println("Server starting on http://localhost:8080")
    http.ListenAndServe(":8080", r)
}
```

---

## Key Concepts Explained

### 1. Middleware Order

```go
r.Use(loggingMiddleware)  // First - logs request
r.Use(corsMiddleware)      // Second - adds CORS headers
r.Use(jsonMiddleware)      // Third - sets Content-Type
```

**Execution Order**:
1. Request comes in
2. loggingMiddleware (starts timer)
3. corsMiddleware (adds headers)
4. jsonMiddleware (sets Content-Type)
5. Handler executes
6. Response flows back through middleware
7. loggingMiddleware (logs duration)

### 2. Capturing Status Code in Middleware

The standard ResponseWriter doesn't expose the status code after it's written. We create a wrapper:

```go
type statusRecorder struct {
    http.ResponseWriter
    status int
}

func (r *statusRecorder) WriteHeader(status int) {
    r.status = status  // Capture it
    r.ResponseWriter.WriteHeader(status)  // Call original
}
```

### 3. Query Parameter Filtering

```go
completedParam := r.URL.Query().Get("completed")

if completedParam == "" {
    // No filter applied
}

isCompleted := completedParam == "true"
if task.Completed == isCompleted {
    // Include this task
}
```

### 4. Validation Pattern

```go
func validateTaskInput(title string) error {
    title = strings.TrimSpace(title)
    if title == "" {
        return errors.New("Title is required")
    }
    if len(title) < 3 {
        return errors.New("Title must be at least 3 characters")
    }
    return nil
}

// Usage
if err := validateTaskInput(req.Title); err != nil {
    respondError(w, http.StatusBadRequest, "validation_failed", err.Error())
    return
}
```

### 5. Proper HTTP Status Codes

```go
// 200 OK - Successful GET, PUT
respondJSON(w, http.StatusOK, task)

// 201 Created - Successful POST
respondJSON(w, http.StatusCreated, task)

// 204 No Content - Successful DELETE (no body)
w.WriteHeader(http.StatusNoContent)

// 400 Bad Request - Validation error
respondError(w, http.StatusBadRequest, "validation_failed", "...")

// 404 Not Found - Resource doesn't exist
respondError(w, http.StatusNotFound, "not_found", "Task not found")
```

---

## Bonus Solutions

### Bonus 1: Statistics Endpoint

```go
type TaskStats struct {
    Total      int `json:"total"`
    Completed  int `json:"completed"`
    Incomplete int `json:"incomplete"`
}

func getTaskStats(w http.ResponseWriter, r *http.Request) {
    tasksMu.RLock()
    defer tasksMu.RUnlock()

    stats := TaskStats{
        Total: len(tasks),
    }

    for _, task := range tasks {
        if task.Completed {
            stats.Completed++
        } else {
            stats.Incomplete++
        }
    }

    respondJSON(w, http.StatusOK, stats)
}

// In main()
r.HandleFunc("/tasks/stats", getTaskStats).Methods("GET")
```

**Important**: Register `/tasks/stats` BEFORE `/tasks/{id}` to avoid "stats" being treated as an ID.

### Bonus 2: Partial Updates (PATCH)

```go
type PatchTaskRequest struct {
    Title       *string `json:"title,omitempty"`
    Description *string `json:"description,omitempty"`
    Completed   *bool   `json:"completed,omitempty"`
}

func patchTask(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        respondError(w, http.StatusBadRequest, "validation_failed", "Invalid task ID")
        return
    }

    var req PatchTaskRequest
    err = json.NewDecoder(r.Body).Decode(&req)
    if err != nil {
        respondError(w, http.StatusBadRequest, "invalid_json", "Invalid JSON format")
        return
    }
    defer r.Body.Close()

    tasksMu.Lock()
    task, exists := tasks[id]
    if !exists {
        tasksMu.Unlock()
        respondError(w, http.StatusNotFound, "not_found", "Task not found")
        return
    }

    // Update only provided fields
    if req.Title != nil {
        if err := validateTaskInput(*req.Title); err != nil {
            tasksMu.Unlock()
            respondError(w, http.StatusBadRequest, "validation_failed", err.Error())
            return
        }
        task.Title = *req.Title
    }

    if req.Description != nil {
        task.Description = *req.Description
    }

    if req.Completed != nil {
        task.Completed = *req.Completed
    }

    tasks[id] = task
    tasksMu.Unlock()

    respondJSON(w, http.StatusOK, task)
}

// In main()
r.HandleFunc("/tasks/{id}", patchTask).Methods("PATCH")
```

**Note**: Using pointers allows us to distinguish between "not provided" and "set to zero value".

### Bonus 3: Bulk Delete

```go
func bulkDeleteTasks(w http.ResponseWriter, r *http.Request) {
    completedParam := r.URL.Query().Get("completed")

    if completedParam != "true" {
        respondError(w, http.StatusBadRequest, "invalid_parameter",
            "Can only bulk delete completed tasks")
        return
    }

    tasksMu.Lock()
    defer tasksMu.Unlock()

    deletedCount := 0
    for id, task := range tasks {
        if task.Completed {
            delete(tasks, id)
            deletedCount++
        }
    }

    respondJSON(w, http.StatusOK, map[string]interface{}{
        "deleted": deletedCount,
        "message": fmt.Sprintf("Deleted %d completed tasks", deletedCount),
    })
}

// In main() - register BEFORE /tasks/{id}
r.HandleFunc("/tasks", bulkDeleteTasks).Methods("DELETE").
    Queries("completed", "true")
```

### Bonus 4: Search

```go
func getTasks(w http.ResponseWriter, r *http.Request) {
    completedParam := r.URL.Query().Get("completed")
    searchQuery := strings.ToLower(r.URL.Query().Get("search"))

    tasksMu.RLock()
    defer tasksMu.RUnlock()

    taskList := []Task{}
    for _, task := range tasks {
        // Filter by completed status
        if completedParam != "" {
            isCompleted := completedParam == "true"
            if task.Completed != isCompleted {
                continue
            }
        }

        // Filter by search query
        if searchQuery != "" {
            titleMatch := strings.Contains(strings.ToLower(task.Title), searchQuery)
            descMatch := strings.Contains(strings.ToLower(task.Description), searchQuery)
            if !titleMatch && !descMatch {
                continue
            }
        }

        taskList = append(taskList, task)
    }

    respondJSON(w, http.StatusOK, taskList)
}
```

### Bonus 5: Sorting

```go
import "sort"

func getTasks(w http.ResponseWriter, r *http.Request) {
    completedParam := r.URL.Query().Get("completed")
    searchQuery := strings.ToLower(r.URL.Query().Get("search"))
    sortBy := r.URL.Query().Get("sort")
    order := r.URL.Query().Get("order")

    tasksMu.RLock()
    defer tasksMu.RUnlock()

    taskList := []Task{}
    for _, task := range tasks {
        // Apply filters...
        taskList = append(taskList, task)
    }

    // Sort
    if sortBy != "" {
        switch sortBy {
        case "title":
            sort.Slice(taskList, func(i, j int) bool {
                if order == "desc" {
                    return taskList[i].Title > taskList[j].Title
                }
                return taskList[i].Title < taskList[j].Title
            })
        case "created_at":
            sort.Slice(taskList, func(i, j int) bool {
                if order == "desc" {
                    return taskList[i].CreatedAt.After(taskList[j].CreatedAt)
                }
                return taskList[i].CreatedAt.Before(taskList[j].CreatedAt)
            })
        }
    }

    respondJSON(w, http.StatusOK, taskList)
}
```

---

## Testing the Solution

### Test Script

Create a file `test.sh`:

```bash
#!/bin/bash

BASE_URL="http://localhost:8080"

echo "=== Creating Tasks ==="
curl -X POST $BASE_URL/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Learn Go","description":"Complete Unit 2"}'
echo ""

curl -X POST $BASE_URL/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Build API","description":"Create REST API"}'
echo ""

curl -X POST $BASE_URL/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Write Tests","description":"Test the API"}'
echo ""

echo "=== Getting All Tasks ==="
curl $BASE_URL/tasks
echo ""

echo "=== Getting Single Task ==="
curl $BASE_URL/tasks/1
echo ""

echo "=== Updating Task ==="
curl -X PUT $BASE_URL/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"title":"Learn Go","description":"Complete Unit 2","completed":true}'
echo ""

echo "=== Filter Completed Tasks ==="
curl "$BASE_URL/tasks?completed=true"
echo ""

echo "=== Filter Incomplete Tasks ==="
curl "$BASE_URL/tasks?completed=false"
echo ""

echo "=== Deleting Task ==="
curl -X DELETE $BASE_URL/tasks/2
echo ""

echo "=== Verify Deletion ==="
curl $BASE_URL/tasks
echo ""

echo "=== Test Validation (should fail) ==="
curl -X POST $BASE_URL/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Go","description":"Too short"}'
echo ""

echo "=== Test 404 (should fail) ==="
curl $BASE_URL/tasks/999
echo ""
```

Run with: `chmod +x test.sh && ./test.sh`

---

## Common Mistakes and Solutions

### Mistake 1: Not Closing Request Body

```go
// WRONG
func createTask(w http.ResponseWriter, r *http.Request) {
    var req CreateTaskRequest
    json.NewDecoder(r.Body).Decode(&req)
    // Body not closed - memory leak!
}

// RIGHT
func createTask(w http.ResponseWriter, r *http.Request) {
    var req CreateTaskRequest
    json.NewDecoder(r.Body).Decode(&req)
    defer r.Body.Close()  // Always close
}
```

### Mistake 2: Setting Headers After Writing Body

```go
// WRONG
func handler(w http.ResponseWriter, r *http.Request) {
    json.NewEncoder(w).Encode(data)
    w.Header().Set("Content-Type", "application/json")  // Too late!
}

// RIGHT
func handler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(data)
}
```

### Mistake 3: Not Handling Invalid JSON

```go
// WRONG
var req CreateTaskRequest
json.NewDecoder(r.Body).Decode(&req)  // Ignoring error

// RIGHT
var req CreateTaskRequest
err := json.NewDecoder(r.Body).Decode(&req)
if err != nil {
    respondError(w, http.StatusBadRequest, "invalid_json", "Invalid JSON format")
    return
}
```

### Mistake 4: Race Conditions with Map

```go
// WRONG - concurrent access without locks
tasks[id] = task  // Race condition!

// RIGHT - use mutex
tasksMu.Lock()
tasks[id] = task
tasksMu.Unlock()
```

---

## What You've Learned

✅ Complete REST API implementation  
✅ Proper HTTP status codes  
✅ Request validation patterns  
✅ Query parameter filtering  
✅ Middleware composition  
✅ Capturing response status  
✅ Thread-safe concurrent access  
✅ Error response patterns  
✅ Testing HTTP endpoints  

You now have a production-ready REST API pattern to follow!
