# API Development with Go
## A Practical Course for Programmers

---

## Course Overview

This course teaches you to build production-ready RESTful APIs using Go (Golang). Designed for developers who already understand programming fundamentals, you'll learn Go from scratch and master essential API concepts including security, versioning, documentation, and performance optimization.

**Duration**: 12-16 hours of instruction + exercises  
**Level**: Intermediate (assumes programming knowledge)  
**Prerequisites**: Understanding of variables, functions, loops, and basic data structures in any language

---

## Why Go for APIs?

Go has become the industry standard for building APIs and microservices at companies like Google, Uber, Netflix, and Dropbox because:

- ✅ **Simple & Explicit** - Less framework magic, clearer code
- ✅ **Fast & Efficient** - Compiled performance with easy concurrency
- ✅ **Built for APIs** - Excellent standard library for HTTP servers
- ✅ **Single Binary Deploy** - No runtime dependencies
- ✅ **Industry Demand** - Cloud-native development standard

---

## Course Structure

### **Unit 1: Go Crash Course** (60 min)
Learn Go fundamentals: syntax, types, structs, error handling, JSON, and more.

**Exercises**:
- Student Grade Manager (structs, slices, functions)
- JSON Config Manager (file I/O, validation, nested data)

---

### **Unit 2: Building HTTP Servers** (60 min)
Create your first HTTP server, handle requests/responses, work with routes and middleware.

**Topics**:
- `net/http` package basics
- Request/response handling
- Routing with Gorilla Mux
- Project structure and organization

**Exercises**:
- Basic REST API for a task manager
- Request validation and error handling

---

### **Unit 3: Authentication & Authorization** (90 min)
Implement secure user authentication and role-based access control.

**Topics**:
- JWT (JSON Web Tokens) implementation
- Authentication middleware
- Password hashing with bcrypt
- Role-based authorization
- Session management

**Exercises**:
- Build login/register system
- Protect API endpoints with JWT
- Implement admin-only routes

---

### **Unit 4: API Versioning** (45 min)
Learn strategies for evolving your API without breaking clients.

**Topics**:
- URL path versioning (v1, v2)
- Header-based versioning
- Content negotiation
- Deprecation strategies
- Backward compatibility

**Exercises**:
- Create v2 of an existing API
- Implement deprecation headers

---

### **Unit 5: API Documentation** (60 min)
Auto-generate interactive documentation with Swagger/OpenAPI.

**Topics**:
- OpenAPI specification
- Swagger annotations in Go
- Using Swag to generate docs
- Documenting endpoints and models
- Interactive API testing with Swagger UI

**Exercises**:
- Document your entire API
- Create example requests/responses
- Add security definitions

---

### **Unit 6: Caching Strategies** (60 min)
Improve API performance with intelligent caching.

**Topics**:
- In-memory caching
- Redis integration
- HTTP cache headers
- ETags for conditional requests
- Cache invalidation patterns

**Exercises**:
- Implement multi-level caching
- Add cache headers to responses
- Build cache invalidation logic

---

### **Unit 7: Pagination** (45 min)
Handle large datasets efficiently with pagination.

**Topics**:
- Offset-based pagination
- Cursor-based pagination
- Link headers (RESTful approach)
- Performance considerations

**Exercises**:
- Implement both pagination strategies
- Add pagination metadata to responses

---

### **Unit 8: Rate Limiting** (60 min)
Protect your API from abuse with rate limiting.

**Topics**:
- Simple in-memory rate limiters
- Token bucket algorithm
- Per-user rate limiting
- Redis-based distributed limiting
- Rate limit headers

**Exercises**:
- Build a rate limiter from scratch
- Implement tiered rate limits (free vs premium users)
- Add proper HTTP headers

---

### **Unit 9: Integration & Best Practices** (90 min)
Bring everything together into a production-ready API.

**Topics**:
- Putting it all together
- Logging and monitoring
- Graceful shutdown
- Testing strategies
- Deployment considerations
- Security best practices

**Final Project**:
- Build a complete API with all features
- Deploy to a cloud platform

---

## What You'll Build

Throughout this course, you'll build increasingly sophisticated APIs:

1. **Simple HTTP Server** - Hello World to basic routing
2. **Task Management API** - CRUD operations with validation
3. **Authenticated User System** - Registration, login, protected routes
4. **Versioned API** - Multiple API versions side-by-side
5. **Documented API** - Full Swagger documentation
6. **High-Performance API** - Caching, pagination, rate limiting
7. **Production-Ready API** - All concepts integrated

---

## Learning Outcomes

By the end of this course, you will be able to:

✅ Write idiomatic Go code  
✅ Build RESTful APIs from scratch  
✅ Implement JWT authentication and authorization  
✅ Version APIs without breaking existing clients  
✅ Generate interactive API documentation  
✅ Optimize API performance with caching  
✅ Handle large datasets with pagination  
✅ Protect APIs with rate limiting  
✅ Follow industry best practices  
✅ Deploy production-ready APIs  

---

## Tools & Technologies

- **Language**: Go 1.21+
- **Router**: Gorilla Mux
- **Auth**: JWT (golang-jwt)
- **Documentation**: Swag (Swagger/OpenAPI)
- **Caching**: In-memory & Redis
- **Database**: PostgreSQL (examples)
- **Testing**: Go's built-in testing
- **Tools**: Postman/Insomnia for API testing

---

## Course Philosophy

### **Learning by Doing**
Every concept is reinforced with hands-on exercises. You'll write code from day one.

### **Practical Over Theoretical**
Focus on real-world patterns used in production APIs, not academic exercises.

### **Progressive Complexity**
Start simple, add features incrementally. Each unit builds on previous knowledge.

### **Industry-Relevant**
Learn patterns and practices used at major tech companies.

---

## Who This Course Is For

✅ **Backend developers** transitioning to Go  
✅ **API developers** wanting to learn modern practices  
✅ **Students** learning professional API development  
✅ **Engineers** building microservices  

❌ Not for: Complete programming beginners (learn fundamentals first)

---

## Getting Started

### Prerequisites Check
- [ ] Comfortable with variables, functions, loops
- [ ] Understand basic data structures (arrays, objects/maps)
- [ ] Familiar with HTTP concepts (GET, POST, status codes)
- [ ] Have Go installed (`go version` to verify)
- [ ] Text editor or IDE ready (VS Code recommended)

### Recommended Setup
- **Editor**: VS Code with Go extension
- **API Client**: Postman or Insomnia
- **Database**: PostgreSQL (optional, examples provided)
- **Redis**: For caching exercises (optional)

---

## Course Materials

Each unit includes:
- 📖 **Lesson Document** - Concepts, code examples, explanations
- 💻 **Exercises** - 2-3 hands-on coding challenges
- 🎯 **Solutions** - Reference implementations
- 🚀 **Bonus Challenges** - For advanced learners

---

## Tips for Success

1. **Type the code yourself** - Don't copy-paste, type it to build muscle memory
2. **Do the exercises** - They're essential, not optional
3. **Experiment** - Break things, fix them, understand why
4. **Read error messages** - Go's errors are helpful, not scary
5. **Build something real** - Apply concepts to your own project ideas

---

## Support & Resources

- **Official Go Docs**: https://go.dev/doc/
- **Go Tour**: https://go.dev/tour/ (quick interactive tutorial)
- **Effective Go**: https://go.dev/doc/effective_go (style guide)
- **Go by Example**: https://gobyexample.com/ (code snippets)

---

## What Makes This Course Different?

Unlike other courses that either teach Go OR APIs, this course:

🎯 **Integrates both** - Learn Go while building real APIs  
🎯 **Goes deep on key concepts** - Not just CRUD, but auth, caching, versioning  
🎯 **Production-focused** - Learn patterns used in real companies  
🎯 **Hands-on throughout** - Write code in every unit  

---

## Ready to Start?

Begin with **Unit 1: Go Crash Course** to learn Go fundamentals, then progress through each unit building increasingly sophisticated APIs.

Each unit is self-contained but builds on previous knowledge, so follow them in order for the best learning experience.

**Let's build something amazing! 🚀**

---

## Course Outline at a Glance

| Unit | Topic | Duration | Key Takeaway |
|------|-------|----------|--------------|
| 1 | Go Crash Course | 60 min | Go fundamentals & syntax |
| 2 | HTTP Servers | 60 min | Building REST endpoints |
| 3 | Auth & Authorization | 90 min | JWT & security |
| 4 | API Versioning | 45 min | Managing API evolution |
| 5 | Documentation | 60 min | Swagger/OpenAPI |
| 6 | Caching | 60 min | Performance optimization |
| 7 | Pagination | 45 min | Handling large datasets |
| 8 | Rate Limiting | 60 min | API protection |
| 9 | Integration | 90 min | Production-ready APIs |

**Total**: ~9.5 hours instruction + ~6.5 hours exercises = **~16 hours**

---

**Start your journey**: Open `unit1_go_crash_course.md` to begin! 🎓
