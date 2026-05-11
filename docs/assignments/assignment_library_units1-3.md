# Assignment: Library Management API (Units 1-3)

**Duration**: 10-15 hours  
**Difficulty**: Intermediate to Advanced  
**Units Covered**: 1-3 (Go fundamentals, HTTP servers, Authentication & Authorization)

---

## 📋 Overview

Build a complete **Library Management API** that facilitates book borrowing and returns with role-based access control. This assignment tests your understanding of:

- ✅ **Unit 1**: Go fundamentals (structs, interfaces, error handling, concurrency)
- ✅ **Unit 2**: HTTP servers (routing, middleware, JSON APIs)
- ✅ **Unit 3**: Authentication & Authorization (JWT, RBAC with 4 roles)

---

## 🎯 Project Description

Develop a RESTful API for a library system where users can search for books, request to borrow them, and librarians can manage the borrowing process. The system must implement a sophisticated 4-role permission system with hierarchical access control.

---

## 📊 Data Models

### User
```go
type User struct {
    ID        int       `json:"id"`
    Email     string    `json:"email"`
    Name      string    `json:"name"`
    Password  string    `json:"-"`
    Roles     []Role    `json:"roles"`  // Users can have multiple roles
    CreatedAt time.Time `json:"created_at"`
}

type Role string

const (
    RoleSuperAdmin Role = "superadmin"
    RoleAdmin      Role = "admin"
    RoleLibrarian  Role = "librarian"
    RoleBorrower   Role = "borrower"
)
```

### Book
```go
type Book struct {
    ID          int       `json:"id"`
    Title       string    `json:"title"`
    Author      string    `json:"author"`
    Genre       string    `json:"genre"`
    ISBN        string    `json:"isbn"`
    Status      BookStatus `json:"status"`
    BorrowedBy  int       `json:"borrowed_by,omitempty"`  // User ID if borrowed
    Archived    bool      `json:"archived"`
    CreatedAt   time.Time `json:"created_at"`
}

type BookStatus string

const (
    StatusAvailable BookStatus = "available"
    StatusBorrowed  BookStatus = "borrowed"
)
```

### BorrowRequest
```go
type BorrowRequest struct {
    ID            int           `json:"id"`
    BookID        int           `json:"book_id"`
    BorrowerID    int           `json:"borrower_id"`
    Status        RequestStatus `json:"status"`
    RequestedAt   time.Time     `json:"requested_at"`
    ProcessedAt   *time.Time    `json:"processed_at,omitempty"`
    ProcessedBy   int           `json:"processed_by,omitempty"`  // Librarian ID
}

type RequestStatus string

const (
    RequestPending  RequestStatus = "pending"
    RequestApproved RequestStatus = "approved"
    RequestDenied   RequestStatus = "denied"
)
```

### BorrowHistory
```go
type BorrowHistory struct {
    ID         int       `json:"id"`
    BookID     int       `json:"book_id"`
    BorrowerID int       `json:"borrower_id"`
    BorrowedAt time.Time `json:"borrowed_at"`
    ReturnedAt *time.Time `json:"returned_at,omitempty"`
    LibrarianID int      `json:"librarian_id"`  // Who approved/returned
}
```

---

## 🛣️ API Endpoints

### Authentication (No role required)
```
POST   /register              - Register new user
POST   /login                 - Login and get JWT token
GET    /profile               - Get current user profile (authenticated)
```

### Books (Public - read only for guests)
```
GET    /api/books                     - List all books (non-archived)
GET    /api/books/{id}                - Get book details
GET    /api/books/search?q={query}    - Search by title, author, or genre
```

### Books (Librarian only)
```
POST   /api/books                     - Add new book
POST   /api/books/{id}/archive        - Archive a book
DELETE /api/books/{id}/unarchive      - Unarchive (SuperAdmin only)
PATCH  /api/books/{id}/return         - Mark book as returned
```

### Borrow Requests (Borrower)
```
POST   /api/requests                  - Create borrow request
GET    /api/requests/my               - Get my requests
DELETE /api/requests/{id}             - Cancel pending request (own only)
```

### Borrow Requests (Librarian)
```
GET    /api/requests                  - List all pending requests
PATCH  /api/requests/{id}/approve     - Approve request
PATCH  /api/requests/{id}/deny        - Deny request
```

### User Management (Admin/SuperAdmin)
```
GET    /api/users                     - List all users
PATCH  /api/users/{id}/roles          - Grant/revoke roles
GET    /api/users/{id}/history        - View user borrow history
```

### My Borrowings (Borrower)
```
GET    /api/my/borrowed               - Books I currently have
GET    /api/my/history                - My borrowing history
```

---

## 🔐 Role-Based Permissions

### **SuperAdmin**
- ✅ **Full system access** - can perform any action
- ✅ Grant/revoke **all roles** (including SuperAdmin and Admin)
- ✅ Unarchive books
- ✅ View all system data and history
- ⚠️ **Special**: When revoking own SuperAdmin role, require confirmation

### **Admin**
- ✅ Manage **Librarian** and **Borrower** roles only
- ✅ Can grant/revoke roles for non-Admin, non-SuperAdmin users
- ❌ Cannot grant/revoke Admin or SuperAdmin roles
- ❌ Does **not** have Librarian or Borrower abilities (unless also has those roles)

### **Librarian**
- ✅ View all pending borrow requests
- ✅ Approve/deny borrow requests
  - Can only approve if book is **available**
  - Approving marks book as borrowed by requester
- ✅ Mark books as **returned**
- ✅ Add new books to library
- ✅ **Archive** books
  - Book becomes invisible to borrowers
  - All pending requests for that book are auto-denied
  - Book remains in database (not deleted)
  - **Only SuperAdmin** can unarchive
- ❌ Cannot modify user roles

### **Borrower**
- ✅ View all books (including status: available/borrowed)
- ✅ Request to borrow any **non-archived** book
  - Can request even if book is currently borrowed
  - Request goes to "pending" status
- ✅ View own current borrowed books
- ✅ View status of own requests
- ✅ Cancel own **pending** requests
- ❌ Cannot see archived books

### **No Role / Guest**
- ✅ View all **non-archived** books
- ❌ Cannot see book status (available/borrowed)
- ❌ Cannot request books
- ❌ Cannot see archived books

---

## 📝 Implementation Requirements

### 1. Project Structure
```
library-api/
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── models/
│   │   ├── user.go
│   │   ├── book.go
│   │   ├── request.go
│   │   └── history.go
│   ├── repository/
│   │   └── repository.go
│   ├── service/
│   │   ├── auth_service.go
│   │   ├── book_service.go
│   │   └── borrow_service.go
│   ├── handler/
│   │   ├── auth_handler.go
│   │   ├── book_handler.go
│   │   ├── request_handler.go
│   │   └── user_handler.go
│   └── middleware/
│       ├── auth.go
│       ├── role.go
│       └── cors.go
├── go.mod
├── go.sum
└── README.md
```

### 2. In-Memory Storage
Use maps with proper synchronization:
```go
type Repository struct {
    users           map[int]*User
    books           map[int]*Book
    requests        map[int]*BorrowRequest
    borrowHistory   []*BorrowHistory
    emailIndex      map[string]int
    mu              sync.RWMutex
}
```

### 3. Authentication Middleware
```go
func AuthMiddleware(authService *AuthService) mux.MiddlewareFunc {
    // Extract and validate JWT
    // Add user info to context
    // Continue to next handler
}
```

### 4. Role Authorization Middleware
```go
func RequireAnyRole(roles ...Role) mux.MiddlewareFunc {
    // Check if user has at least one of the required roles
}

func RequireAllRoles(roles ...Role) mux.MiddlewareFunc {
    // Check if user has all required roles
}
```

### 5. Business Logic Requirements

**Book Archiving**:
```go
func (s *BookService) ArchiveBook(bookID, librarianID int) error {
    // 1. Mark book as archived
    // 2. Auto-deny all pending requests for this book
    // 3. Record librarian who archived
    // 4. Return error if book is currently borrowed
}
```

**Request Approval**:
```go
func (s *BorrowService) ApproveRequest(requestID, librarianID int) error {
    // 1. Check request is pending
    // 2. Check book is available (not borrowed)
    // 3. Mark book as borrowed
    // 4. Update request status to approved
    // 5. Create borrow history record
    // 6. Record librarian who approved
}
```

**Book Return**:
```go
func (s *BorrowService) ReturnBook(bookID, librarianID int) error {
    // 1. Check book is currently borrowed
    // 2. Mark book as available
    // 3. Update borrow history with return date
    // 4. Record librarian who processed return
}
```

**Role Management with Special Handling**:
```go
func (s *UserService) RevokeRole(targetUserID, adminID int, role Role) error {
    // 1. Check admin has permission to revoke this role
    // 2. If revoking own SuperAdmin role, require confirmation flag
    // 3. Remove role from user
    // 4. Return appropriate error for permission issues
}
```

---

## ✅ Acceptance Criteria

### Functionality
- [ ] Users can register with multiple roles
- [ ] JWT authentication works correctly
- [ ] Books can be searched by title, author, or genre
- [ ] Borrowers can request books
- [ ] Librarians can approve/deny requests (only if book available)
- [ ] Librarians can mark books as returned
- [ ] Librarians can archive books (auto-denies pending requests)
- [ ] SuperAdmin can unarchive books
- [ ] Admin can manage Librarian/Borrower roles only
- [ ] SuperAdmin can manage all roles with confirmation for self-revocation
- [ ] Guests can view books but not status
- [ ] All role permissions enforced correctly

### Code Quality
- [ ] Clean code structure with proper separation
- [ ] Thread-safe operations (mutex locks)
- [ ] Proper error handling
- [ ] Input validation
- [ ] Password hashing (bcrypt)
- [ ] JWT token validation
- [ ] RESTful API design

### Business Rules
- [ ] Books can only be borrowed if available
- [ ] Archived books invisible to borrowers
- [ ] Archiving auto-denies pending requests
- [ ] History preserved (books not deleted)
- [ ] Multiple roles per user supported
- [ ] Role hierarchy enforced

---

## 🧪 Testing Guide

### 1. Setup Test Users

```bash
# SuperAdmin
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "super@library.com",
    "password": "super123",
    "name": "Super Admin",
    "roles": ["superadmin", "librarian", "borrower"]
  }'

# Admin
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@library.com",
    "password": "admin123",
    "name": "Admin User",
    "roles": ["admin"]
  }'

# Librarian
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "librarian@library.com",
    "password": "lib123",
    "name": "Librarian User",
    "roles": ["librarian"]
  }'

# Borrower
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "borrower@library.com",
    "password": "borrow123",
    "name": "Borrower User",
    "roles": ["borrower"]
  }'

# Guest (no roles)
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "guest@library.com",
    "password": "guest123",
    "name": "Guest User",
    "roles": []
  }'
```

### 2. Add Books (as Librarian)

```bash
# Login as librarian
curl -X POST http://localhost:8080/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "librarian@library.com",
    "password": "lib123"
  }'

TOKEN="your-token-here"

# Add books
curl -X POST http://localhost:8080/api/books \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "The Go Programming Language",
    "author": "Alan Donovan",
    "genre": "Programming",
    "isbn": "978-0134190440"
  }'

curl -X POST http://localhost:8080/api/books \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Clean Code",
    "author": "Robert Martin",
    "genre": "Programming",
    "isbn": "978-0132350884"
  }'
```

### 3. Test Borrowing Workflow

```bash
# Login as borrower
curl -X POST http://localhost:8080/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "borrower@library.com",
    "password": "borrow123"
  }'

BORROWER_TOKEN="borrower-token"

# Search for books
curl "http://localhost:8080/api/books/search?q=Go" \
  -H "Authorization: Bearer $BORROWER_TOKEN"

# Request to borrow
curl -X POST http://localhost:8080/api/requests \
  -H "Authorization: Bearer $BORROWER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "book_id": 1
  }'

# Check my requests
curl http://localhost:8080/api/requests/my \
  -H "Authorization: Bearer $BORROWER_TOKEN"
```

### 4. Approve Request (as Librarian)

```bash
# Login as librarian
LIBRARIAN_TOKEN="librarian-token"

# View pending requests
curl http://localhost:8080/api/requests \
  -H "Authorization: Bearer $LIBRARIAN_TOKEN"

# Approve request
curl -X PATCH http://localhost:8080/api/requests/1/approve \
  -H "Authorization: Bearer $LIBRARIAN_TOKEN"

# Deny a request
curl -X PATCH http://localhost:8080/api/requests/2/deny \
  -H "Authorization: Bearer $LIBRARIAN_TOKEN"
```

### 5. Archive Book (as Librarian)

```bash
# Archive book (auto-denies pending requests)
curl -X POST http://localhost:8080/api/books/3/archive \
  -H "Authorization: Bearer $LIBRARIAN_TOKEN"

# Book now invisible to borrowers
curl http://localhost:8080/api/books \
  -H "Authorization: Bearer $BORROWER_TOKEN"
# Book 3 should not appear
```

### 6. Role Management (as Admin)

```bash
# Login as admin
ADMIN_TOKEN="admin-token"

# Grant librarian role to a user
curl -X PATCH http://localhost:8080/api/users/5/roles \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "action": "grant",
    "role": "librarian"
  }'

# Try to grant admin role (should fail - admins can't grant admin)
curl -X PATCH http://localhost:8080/api/users/5/roles \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "action": "grant",
    "role": "admin"
  }'
# Should return 403 Forbidden
```

### 7. SuperAdmin Special Actions

```bash
SUPER_TOKEN="super-token"

# Unarchive a book (only SuperAdmin can do this)
curl -X DELETE http://localhost:8080/api/books/3/unarchive \
  -H "Authorization: Bearer $SUPER_TOKEN"

# Revoke own SuperAdmin role (requires confirmation)
curl -X PATCH http://localhost:8080/api/users/1/roles \
  -H "Authorization: Bearer $SUPER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "action": "revoke",
    "role": "superadmin",
    "confirm": true
  }'
```

---

## 🎯 Grading Rubric (100 points)

### Functionality (60 points)
- [ ] Authentication system (10 points)
  - Registration with multiple roles
  - Login with JWT
  - Token validation
- [ ] Book management (10 points)
  - CRUD operations
  - Search functionality
  - Archive/unarchive
- [ ] Borrow request workflow (15 points)
  - Create requests
  - Approve/deny (with availability check)
  - Auto-deny on archive
- [ ] Book return process (5 points)
  - Mark as returned
  - Update availability
- [ ] Role-based permissions (20 points)
  - All 4 roles implemented correctly
  - Permission checks enforced
  - Role hierarchy (Admin vs SuperAdmin)
  - Special SuperAdmin confirmation

### Code Quality (25 points)
- [ ] Project structure (5 points)
  - Clean organization
  - Proper package separation
- [ ] Code quality (10 points)
  - Readable, maintainable code
  - Error handling
  - Input validation
- [ ] Security (10 points)
  - Password hashing
  - JWT implementation
  - Thread safety

### Business Logic (15 points)
- [ ] Correct business rules (10 points)
  - Book availability checks
  - Archive behavior
  - Request lifecycle
- [ ] Data integrity (5 points)
  - History preservation
  - Consistent state

---

## 🌟 Bonus Challenges (+20 points)

### 1. Audit Trail System (+10 points)
Track all administrative actions:
```go
type AuditLog struct {
    ID        int
    Action    string  // "role_granted", "role_revoked", "request_approved", etc.
    PerformedBy int    // User ID
    TargetUser  int
    TargetBook  int
    Details   string
    Timestamp time.Time
}

// Endpoint
GET /api/admin/audit  // SuperAdmin only
```

### 2. Enhanced Search (+3 points)
```go
// Search with multiple filters
GET /api/books/search?title=Go&author=Donovan&genre=Programming&status=available
```

### 3. Request Denial with Reason (+2 points)
```go
type DenyRequest struct {
    Reason string `json:"reason"`
}

PATCH /api/requests/{id}/deny
```

### 4. Book Borrowing Queue (+5 points)
```go
// If book borrowed, allow queueing
// Auto-approve next in queue when returned
POST /api/books/{id}/queue
GET  /api/books/{id}/queue  // See queue position
```

---

## 💡 Implementation Tips

### 1. Start with Authentication
Build the auth system first as everything depends on it:
1. User registration with multiple roles
2. Password hashing
3. JWT generation
4. Auth middleware
5. Role checking middleware

### 2. Build Core Features
1. Book CRUD (Librarian only)
2. Guest book viewing
3. Borrower requesting
4. Librarian approval workflow

### 3. Add Special Behaviors
1. Archive auto-deny logic
2. SuperAdmin confirmation
3. Admin role restrictions

### 4. Role Permission Helper
```go
func (u *User) HasRole(role Role) bool {
    for _, r := range u.Roles {
        if r == role {
            return true
        }
    }
    return false
}

func (u *User) HasAnyRole(roles ...Role) bool {
    for _, role := range roles {
        if u.HasRole(role) {
            return true
        }
    }
    return false
}
```

### 5. Archive Book Pattern
```go
func (s *BookService) ArchiveBook(bookID, librarianID int) error {
    book, err := s.repo.GetBook(bookID)
    if err != nil {
        return err
    }
    
    if book.Status == StatusBorrowed {
        return errors.New("cannot archive borrowed book")
    }
    
    // Mark as archived
    book.Archived = true
    
    // Auto-deny pending requests
    s.repo.DenyPendingRequestsForBook(bookID, librarianID, "book archived")
    
    return s.repo.UpdateBook(book)
}
```

---

## 📚 Expected Learning Outcomes

After completing this assignment, you will be able to:

✅ Build complex multi-role authorization systems  
✅ Implement JWT authentication with role-based access  
✅ Design RESTful APIs with proper HTTP methods  
✅ Handle complex business logic and state transitions  
✅ Work with concurrent data structures safely  
✅ Validate and sanitize user input  
✅ Structure large Go projects professionally  
✅ Implement hierarchical permission systems  
✅ Manage database-like operations in memory  

---

## 🎓 Summary

This assignment challenges you to build a **production-grade Library Management API** with:
- 4-role permission system with hierarchy
- Complex business logic (archiving, auto-denial, confirmations)
- JWT authentication
- Thread-safe operations
- RESTful API design

**Estimated time**: 10-15 hours

**Good luck building your library system!** 📚🚀
