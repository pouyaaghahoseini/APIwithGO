# Assignment: Task Management API (Units 1-5)

**Duration**: 8-12 hours  
**Difficulty**: Intermediate  
**Units Covered**: 1-5 (Go fundamentals, HTTP servers, Authentication, API Versioning, Documentation)

---

## 📋 Overview

Build a complete **Task Management API** similar to Todoist or Asana. This assignment tests your understanding of:
- ✅ **Unit 1**: Go fundamentals (structs, interfaces, error handling)
- ✅ **Unit 2**: HTTP servers (routing, middleware, JSON)
- ✅ **Unit 3**: Authentication & Authorization (JWT, RBAC)
- ✅ **Unit 4**: API Versioning (v1 and v2 with different features)
- ✅ **Unit 5**: API Documentation (Swagger/OpenAPI)

---

## 🎯 Requirements

### Core Features

1. **User Management**
   - Registration
   - Login
   - User profiles
   - Role-based access (Admin, Manager, Member)

2. **Task Management**
   - Create, read, update, delete tasks
   - Assign tasks to users
   - Set task status (todo, in_progress, done)
   - Set task priority (low, medium, high)
   - Due dates

3. **Project Management**
   - Create projects
   - Add tasks to projects
   - List all tasks in a project
   - Only project owner can delete project

4. **Authentication & Authorization**
   - JWT-based authentication
   - Role-based permissions:
     - **Admin**: Full access to everything
     - **Manager**: Can create projects, assign tasks
     - **Member**: Can only update their own tasks

5. **API Versioning**
   - **v1**: Basic task CRUD
   - **v2**: Enhanced with projects, assignments, and filtering

6. **Documentation**
   - Complete Swagger/OpenAPI documentation
   - All endpoints documented with examples

---

## 📊 Data Models

### User
```go
type User struct {
    ID        int       `json:"id"`
    Email     string    `json:"email"`
    Name      string    `json:"name"`
    Password  string    `json:"-"` // Don't return in JSON
    Role      UserRole  `json:"role"`
    CreatedAt time.Time `json:"created_at"`
}

type UserRole string

const (
    RoleAdmin   UserRole = "admin"
    RoleManager UserRole = "manager"
    RoleMember  UserRole = "member"
)
```

### Task
```go
type Task struct {
    ID          int        `json:"id"`
    Title       string     `json:"title"`
    Description string     `json:"description"`
    Status      TaskStatus `json:"status"`
    Priority    Priority   `json:"priority"`
    AssignedTo  int        `json:"assigned_to,omitempty"` // User ID
    ProjectID   int        `json:"project_id,omitempty"`  // v2 only
    DueDate     time.Time  `json:"due_date,omitempty"`
    CreatedBy   int        `json:"created_by"`
    CreatedAt   time.Time  `json:"created_at"`
    UpdatedAt   time.Time  `json:"updated_at"`
}

type TaskStatus string

const (
    StatusTodo       TaskStatus = "todo"
    StatusInProgress TaskStatus = "in_progress"
    StatusDone       TaskStatus = "done"
)

type Priority string

const (
    PriorityLow    Priority = "low"
    PriorityMedium Priority = "medium"
    PriorityHigh   Priority = "high"
)
```

### Project (v2 only)
```go
type Project struct {
    ID          int       `json:"id"`
    Name        string    `json:"name"`
    Description string    `json:"description"`
    OwnerID     int       `json:"owner_id"`
    CreatedAt   time.Time `json:"created_at"`
}
```

---

## 🛣️ API Endpoints

### Authentication (No versioning needed)
```
POST   /register          - Register new user
POST   /login             - Login and get JWT token
GET    /profile           - Get current user profile (authenticated)
```

### API v1 - Basic Task Management

```
POST   /api/v1/tasks                    - Create task
GET    /api/v1/tasks                    - List all tasks
GET    /api/v1/tasks/{id}               - Get task by ID
PUT    /api/v1/tasks/{id}               - Update task
DELETE /api/v1/tasks/{id}               - Delete task
GET    /api/v1/tasks?status=todo        - Filter by status
GET    /api/v1/tasks?assigned_to=5      - Filter by assignee
```

### API v2 - Enhanced with Projects

```
# Tasks (enhanced)
POST   /api/v2/tasks                    - Create task (with project_id)
GET    /api/v2/tasks                    - List tasks (more filters)
GET    /api/v2/tasks/{id}               - Get task with project info
PUT    /api/v2/tasks/{id}               - Update task
DELETE /api/v2/tasks/{id}               - Delete task
PATCH  /api/v2/tasks/{id}/assign        - Assign task to user
PATCH  /api/v2/tasks/{id}/status        - Update task status
GET    /api/v2/tasks?project_id=3       - Filter by project
GET    /api/v2/tasks?priority=high      - Filter by priority

# Projects (v2 only)
POST   /api/v2/projects                 - Create project
GET    /api/v2/projects                 - List all projects
GET    /api/v2/projects/{id}            - Get project by ID
PUT    /api/v2/projects/{id}            - Update project
DELETE /api/v2/projects/{id}            - Delete project
GET    /api/v2/projects/{id}/tasks      - Get all tasks in project
```

### Admin Only (v2)
```
GET    /api/v2/admin/users              - List all users
DELETE /api/v2/admin/users/{id}         - Delete user
PATCH  /api/v2/admin/users/{id}/role    - Change user role
GET    /api/v2/admin/stats              - Get system statistics
```

---

## 🔐 Authentication & Authorization

### JWT Token Structure
```json
{
  "user_id": 1,
  "email": "user@example.com",
  "role": "manager",
  "exp": 1234567890
}
```

### Permission Matrix

| Action | Admin | Manager | Member |
|--------|-------|---------|--------|
| Create task | ✅ | ✅ | ✅ |
| View all tasks | ✅ | ✅ | ❌ (own only) |
| Update any task | ✅ | ✅ | ❌ (own only) |
| Delete any task | ✅ | ✅ | ❌ (own only) |
| Create project | ✅ | ✅ | ❌ |
| Delete project | ✅ | ✅ (own) | ❌ |
| Assign tasks | ✅ | ✅ | ❌ |
| Manage users | ✅ | ❌ | ❌ |

---

## 📝 Implementation Requirements

### 1. Project Structure
```
task-api/
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── models/
│   │   ├── user.go
│   │   ├── task.go
│   │   └── project.go
│   ├── repository/
│   │   ├── user_repository.go
│   │   ├── task_repository.go
│   │   └── project_repository.go
│   ├── service/
│   │   ├── auth_service.go
│   │   ├── task_service.go
│   │   └── project_service.go
│   ├── handler/
│   │   ├── auth_handler.go
│   │   ├── v1/
│   │   │   └── task_handler.go
│   │   └── v2/
│   │       ├── task_handler.go
│   │       ├── project_handler.go
│   │       └── admin_handler.go
│   └── middleware/
│       ├── auth.go
│       └── role.go
├── docs/
│   └── swagger.yaml
├── go.mod
└── README.md
```

### 2. In-Memory Storage
Use maps for data storage (no database required):
```go
type InMemoryStore struct {
    users    map[int]*User
    tasks    map[int]*Task
    projects map[int]*Project
    nextUserID    int
    nextTaskID    int
    nextProjectID int
    mu       sync.RWMutex
}
```

### 3. Middleware Requirements

**Authentication Middleware**:
```go
func AuthMiddleware(jwtSecret string) mux.MiddlewareFunc {
    // Extract JWT from Authorization header
    // Validate token
    // Add user info to context
    // Call next handler
}
```

**Role Authorization Middleware**:
```go
func RequireRole(roles ...UserRole) mux.MiddlewareFunc {
    // Get user from context
    // Check if user has required role
    // Allow or deny
}
```

### 4. API Versioning
Implement using URL path versioning:
```go
// v1 router
v1 := r.PathPrefix("/api/v1").Subrouter()
v1.HandleFunc("/tasks", v1Handler.CreateTask).Methods("POST")

// v2 router
v2 := r.PathPrefix("/api/v2").Subrouter()
v2.HandleFunc("/tasks", v2Handler.CreateTask).Methods("POST")
```

### 5. Swagger Documentation
Create complete OpenAPI 3.0 specification with:
- All endpoints documented
- Request/response schemas
- Authentication schemes
- Example requests and responses

---

## ✅ Acceptance Criteria

### Functionality
- [ ] Users can register and login
- [ ] JWT tokens are issued on login
- [ ] All v1 endpoints work correctly
- [ ] All v2 endpoints work correctly
- [ ] Role-based permissions enforced
- [ ] Admin can manage users
- [ ] Managers can create projects and assign tasks
- [ ] Members can only update their own tasks
- [ ] Tasks can be filtered by status, priority, assignee, project
- [ ] Projects can be created and deleted by owners

### Code Quality
- [ ] Clean code structure with separation of concerns
- [ ] Proper error handling
- [ ] Input validation
- [ ] Password hashing (bcrypt)
- [ ] JWT token validation
- [ ] Thread-safe operations (mutex locks)
- [ ] Meaningful variable and function names

### Documentation
- [ ] Complete Swagger/OpenAPI documentation
- [ ] README with setup instructions
- [ ] API examples in documentation
- [ ] Environment variables documented

### Testing (Manual)
- [ ] Can register multiple users with different roles
- [ ] Can login and receive valid JWT
- [ ] Can create tasks in both v1 and v2
- [ ] Can create projects in v2
- [ ] Can assign tasks to users
- [ ] Permission checks work correctly
- [ ] Admin endpoints only accessible to admins
- [ ] Filtering works for all parameters

---

## 📦 Deliverables

1. **Source Code**
   - Complete Go project
   - Well-organized directory structure
   - All required files

2. **Documentation**
   - `README.md` with:
     - Setup instructions
     - How to run the API
     - Example API calls
   - `docs/swagger.yaml`:
     - Complete OpenAPI specification
     - Can be viewed in Swagger UI

3. **Test Data**
   - Seed script or function to create:
     - 3 users (one admin, one manager, one member)
     - 10 tasks with various statuses
     - 3 projects

---

## 🧪 Testing Guide

### Manual Testing Steps

**1. Setup and Registration**
```bash
# Start the API
go run cmd/api/main.go

# Register admin user
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@example.com",
    "password": "admin123",
    "name": "Admin User",
    "role": "admin"
  }'

# Register manager
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "manager@example.com",
    "password": "manager123",
    "name": "Manager User",
    "role": "manager"
  }'

# Register member
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "member@example.com",
    "password": "member123",
    "name": "Member User",
    "role": "member"
  }'
```

**2. Authentication**
```bash
# Login as manager
curl -X POST http://localhost:8080/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "manager@example.com",
    "password": "manager123"
  }'

# Save the JWT token from response
TOKEN="eyJhbGc..."
```

**3. Create Tasks (v1)**
```bash
# Create task
curl -X POST http://localhost:8080/api/v1/tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "title": "Complete assignment",
    "description": "Finish the task API",
    "status": "todo",
    "priority": "high",
    "due_date": "2024-12-31T23:59:59Z"
  }'

# List all tasks
curl http://localhost:8080/api/v1/tasks \
  -H "Authorization: Bearer $TOKEN"

# Filter by status
curl "http://localhost:8080/api/v1/tasks?status=todo" \
  -H "Authorization: Bearer $TOKEN"
```

**4. Projects (v2)**
```bash
# Create project
curl -X POST http://localhost:8080/api/v2/projects \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "name": "Assignment Project",
    "description": "All assignment tasks"
  }'

# Create task in project
curl -X POST http://localhost:8080/api/v2/tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "title": "Task in project",
    "project_id": 1,
    "status": "todo",
    "priority": "medium"
  }'

# List project tasks
curl http://localhost:8080/api/v2/projects/1/tasks \
  -H "Authorization: Bearer $TOKEN"
```

**5. Assignment (v2)**
```bash
# Assign task to user
curl -X PATCH http://localhost:8080/api/v2/tasks/1/assign \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "assigned_to": 3
  }'
```

**6. Admin Operations (v2)**
```bash
# Login as admin
curl -X POST http://localhost:8080/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@example.com",
    "password": "admin123"
  }'

ADMIN_TOKEN="eyJhbGc..."

# List all users
curl http://localhost:8080/api/v2/admin/users \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Get statistics
curl http://localhost:8080/api/v2/admin/stats \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Change user role
curl -X PATCH http://localhost:8080/api/v2/admin/users/3/role \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{
    "role": "manager"
  }'
```

---

## 💡 Implementation Tips

### 1. Start with Auth
Build authentication first as all other endpoints depend on it:
```go
// 1. Implement user registration
// 2. Implement password hashing
// 3. Implement login with JWT generation
// 4. Implement auth middleware
// 5. Test with simple protected endpoint
```

### 2. Build v1 First
Get basic task CRUD working before adding complexity:
```go
// 1. Implement in-memory task repository
// 2. Implement task handlers
// 3. Test all v1 endpoints
// 4. Then move to v2
```

### 3. Add Authorization Gradually
```go
// 1. Start with any authenticated user can do anything
// 2. Add role checks to specific endpoints
// 3. Add ownership checks (user can only update their tasks)
// 4. Add admin-only endpoints
```

### 4. Version Differences
```go
// v1: Basic task model
type TaskV1 struct {
    ID          int
    Title       string
    Description string
    Status      string
}

// v2: Enhanced with projects
type TaskV2 struct {
    ID          int
    Title       string
    Description string
    Status      string
    ProjectID   int    // NEW
    Priority    string // NEW
    AssignedTo  int    // NEW
}
```

### 5. Swagger Tips
```yaml
# Start with basic structure
openapi: 3.0.0
info:
  title: Task Management API
  version: 2.0.0

# Add endpoints one by one
paths:
  /register:
    post:
      summary: Register new user
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RegisterRequest'
```

---

## 🎯 Grading Rubric (100 points)

### Functionality (50 points)
- [ ] Authentication works (10 points)
  - Registration creates users
  - Login returns valid JWT
  - Protected endpoints require auth
- [ ] v1 endpoints work (10 points)
  - All CRUD operations
  - Filtering by status and assignee
- [ ] v2 endpoints work (15 points)
  - Projects CRUD
  - Task assignment
  - Project filtering
  - Enhanced task features
- [ ] Authorization works (10 points)
  - Role-based permissions enforced
  - Admin-only endpoints protected
  - Ownership checks work
- [ ] Error handling (5 points)
  - Proper HTTP status codes
  - Meaningful error messages

### Code Quality (30 points)
- [ ] Project structure (10 points)
  - Clean separation of concerns
  - Proper package organization
  - Models, repositories, services, handlers separated
- [ ] Code quality (10 points)
  - Readable code
  - Proper error handling
  - Input validation
  - Thread safety (mutex usage)
- [ ] Security (10 points)
  - Password hashing
  - JWT validation
  - No sensitive data in responses

### Documentation (20 points)
- [ ] README (5 points)
  - Clear setup instructions
  - API examples
  - Environment variables
- [ ] Swagger/OpenAPI (15 points)
  - All endpoints documented
  - Request/response schemas defined
  - Examples provided
  - Can be viewed in Swagger UI

---

## 🌟 Bonus Challenges (+20 points)

### 1. Task Comments (+5 points)
```go
type Comment struct {
    ID        int
    TaskID    int
    UserID    int
    Content   string
    CreatedAt time.Time
}

// Endpoints
POST   /api/v2/tasks/{id}/comments
GET    /api/v2/tasks/{id}/comments
DELETE /api/v2/tasks/{id}/comments/{comment_id}
```

### 2. Task Dependencies (+5 points)
```go
type Task struct {
    // ... existing fields
    BlockedBy []int `json:"blocked_by"` // Task IDs
}

// Can't mark task as done if blocked by incomplete tasks
```

### 3. Activity Log (+5 points)
```go
type Activity struct {
    ID        int
    UserID    int
    Action    string  // "created", "updated", "deleted"
    Resource  string  // "task", "project"
    ResourceID int
    Details   string
    CreatedAt time.Time
}

// Endpoint
GET /api/v2/activity  // Recent activity feed
```

### 4. Task Templates (+5 points)
```go
type Template struct {
    ID    int
    Name  string
    Tasks []TaskTemplate
}

// Create project from template
POST /api/v2/projects/from-template/{template_id}
```

---

## 📚 Resources

### Go Packages You'll Need
```go
import (
    "github.com/gorilla/mux"           // Routing
    "github.com/golang-jwt/jwt/v5"     // JWT
    "golang.org/x/crypto/bcrypt"       // Password hashing
    "github.com/swaggo/http-swagger"   // Swagger UI (optional)
)
```

### Swagger Editor
- Online: https://editor.swagger.io
- VS Code extension: "OpenAPI (Swagger) Editor"

### Testing Tools
- Postman: Create collection of API requests
- curl: Command-line testing
- HTTPie: Better curl alternative

---

## ⏰ Suggested Timeline

**Day 1-2** (4 hours):
- Set up project structure
- Implement authentication (register, login, JWT)
- Create auth middleware

**Day 3-4** (4 hours):
- Implement v1 task endpoints
- Test all CRUD operations
- Add filtering

**Day 5-6** (4 hours):
- Implement v2 with projects
- Add task assignment
- Implement role-based permissions

**Day 7-8** (2-4 hours):
- Write Swagger documentation
- Test all endpoints thoroughly
- Write README

---

## 🎓 Learning Outcomes

After completing this assignment, you will be able to:

✅ Build a complete REST API in Go  
✅ Implement JWT authentication  
✅ Create role-based authorization  
✅ Version APIs properly  
✅ Document APIs with Swagger/OpenAPI  
✅ Handle concurrent access safely  
✅ Structure a Go project professionally  
✅ Validate input and handle errors  
✅ Work with middleware patterns  

**Good luck! This assignment will solidify everything you've learned in Units 1-5!** 🚀
