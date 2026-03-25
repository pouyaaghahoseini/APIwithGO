# Unit 1: Go Crash Course
  
**Prerequisites**: Understanding of programming concepts (variables, functions, loops, data structures)  
**Goal**: Learn Go syntax and fundamentals to build HTTP APIs

---

## 1.1 Why Go?

Go was created at Google in 2009 to solve real problems in large-scale systems:
- **Simple**: Easy to learn, easy to read, easy to maintain
- **Fast**: Compiled to machine code, garbage collected, efficient
- **Concurrent**: Built-in support for concurrent programming (goroutines)
- **Productive**: Fast compilation, great tooling, single binary deployment

**Go's Philosophy**: Simplicity over cleverness. If there are multiple ways to do something, Go usually provides one clear way.

---

## 1.2 Setup and Hello World

### Installation
- Download from: https://go.dev/dl/
- Verify: `go version`

### Hello World

```go
// hello.go
package main

import "fmt"

func main() {
    fmt.Println("Hello, Go!")
}
```

**Run it**:
```bash
go run hello.go
```

**Compile it**:
```bash
go build hello.go
./hello  # On Unix/Mac
hello.exe  # On Windows
```

### Key Observations
- Every Go file starts with a `package` declaration
- `package main` is special - it's the entry point
- `import` brings in other packages
- `func main()` is where execution starts
- No semicolons needed (Go adds them automatically)

---

## 1.3 Variables and Types

### Variable Declaration

```go
package main

import "fmt"

func main() {
    // Explicit type declaration
    var name string = "Alice"
    var age int = 30
    var isActive bool = true
    
    // Type inference (Go figures out the type)
    var city = "New York"
    
    // Short declaration (most common, only inside functions)
    email := "alice@example.com"
    count := 42
    price := 19.99
    
    fmt.Println(name, age, isActive, city, email, count, price)
}
```

### Basic Types

```go
// Integers
var i int = 42           // Platform dependent (32 or 64 bit)
var i8 int8 = 127        // -128 to 127
var i16 int16 = 32767
var i32 int32 = 2147483647
var i64 int64 = 9223372036854775807
var ui uint = 42         // Unsigned (positive only)

// Floats
var f32 float32 = 3.14
var f64 float64 = 3.14159265359

// String
var s string = "Hello"

// Boolean
var b bool = true

// Rune (character, actually int32)
var r rune = 'A'  // Single quotes for runes

// Byte (alias for uint8)
var by byte = 65
```

### Constants

```go
const Pi = 3.14159
const MaxUsers = 100
const AppName = "MyAPI"

// Grouped constants
const (
    StatusPending = "pending"
    StatusActive = "active"
    StatusInactive = "inactive"
)
```

### Zero Values

In Go, variables without explicit initialization get "zero values":

```go
var i int        // 0
var f float64    // 0.0
var b bool       // false
var s string     // "" (empty string)
var p *int       // nil (null pointer)
```

---

## 1.4 Functions

### Basic Functions

```go
// Simple function
func greet(name string) {
    fmt.Println("Hello,", name)
}

// Function with return value
func add(a int, b int) int {
    return a + b
}

// Shorter syntax when parameters have same type
func multiply(a, b int) int {
    return a * b
}

// Multiple return values (very common in Go!)
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}

// Named return values
func calculate(a, b int) (sum int, product int) {
    sum = a + b
    product = a * b
    return  // "naked return" - returns named values
}

func main() {
    greet("Alice")
    
    result := add(5, 3)
    fmt.Println(result)
    
    // Handling multiple return values
    quotient, err := divide(10, 2)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println("Result:", quotient)
    }
    
    s, p := calculate(4, 5)
    fmt.Println("Sum:", s, "Product:", p)
}
```

### Variadic Functions

```go
// Takes variable number of arguments
func sum(numbers ...int) int {
    total := 0
    for _, num := range numbers {
        total += num
    }
    return total
}

func main() {
    fmt.Println(sum(1, 2, 3))           // 6
    fmt.Println(sum(1, 2, 3, 4, 5))     // 15
}
```

---

## 1.5 Control Flow

### If Statements

```go
age := 20

if age >= 18 {
    fmt.Println("Adult")
} else {
    fmt.Println("Minor")
}

// If with initialization statement
if score := getScore(); score >= 90 {
    fmt.Println("Grade: A")
} else if score >= 80 {
    fmt.Println("Grade: B")
} else {
    fmt.Println("Grade: C")
}
// Note: 'score' is only available inside the if/else block
```

### Switch Statements

```go
day := "Monday"

switch day {
case "Monday":
    fmt.Println("Start of work week")
case "Friday":
    fmt.Println("TGIF!")
case "Saturday", "Sunday":  // Multiple values
    fmt.Println("Weekend!")
default:
    fmt.Println("Midweek day")
}

// Switch without expression (cleaner than if-else chains)
score := 85
switch {
case score >= 90:
    fmt.Println("A")
case score >= 80:
    fmt.Println("B")
case score >= 70:
    fmt.Println("C")
default:
    fmt.Println("F")
}
```

### For Loops (Go's only loop!)

```go
// Traditional for loop
for i := 0; i < 5; i++ {
    fmt.Println(i)
}

// While-style loop
count := 0
for count < 5 {
    fmt.Println(count)
    count++
}

// Infinite loop
for {
    fmt.Println("Forever")
    break  // Use break to exit
}

// Range over collections (covered more later)
numbers := []int{1, 2, 3, 4, 5}
for index, value := range numbers {
    fmt.Println(index, value)
}

// Ignore index with _
for _, value := range numbers {
    fmt.Println(value)
}
```

---

## 1.6 Data Structures

### Arrays (Fixed size)

```go
// Array declaration
var arr [5]int  // Array of 5 integers, all zeros
arr[0] = 10
arr[1] = 20

// Array literal
numbers := [5]int{1, 2, 3, 4, 5}

// Auto-size
scores := [...]int{90, 85, 88}  // Size determined by elements

fmt.Println(arr[0])      // Access element
fmt.Println(len(numbers)) // Length
```

### Slices (Dynamic arrays - most common!)

```go
// Slice declaration
var s []int  // Empty slice

// Make a slice with make()
s = make([]int, 5)      // Length 5, all zeros
s = make([]int, 0, 10)  // Length 0, capacity 10

// Slice literal
fruits := []string{"apple", "banana", "orange"}

// Append elements (slices can grow!)
fruits = append(fruits, "grape")
fruits = append(fruits, "mango", "kiwi")

// Slice operations
slice := []int{0, 1, 2, 3, 4, 5}
fmt.Println(slice[1:4])   // [1 2 3] - from index 1 to 3
fmt.Println(slice[:3])    // [0 1 2] - from start to 2
fmt.Println(slice[3:])    // [3 4 5] - from 3 to end
fmt.Println(slice[:])     // [0 1 2 3 4 5] - entire slice

// Length and capacity
fmt.Println(len(fruits))  // Number of elements
fmt.Println(cap(fruits))  // Capacity (underlying array size)
```

### Maps (Hash tables / Dictionaries)

```go
// Map declaration
var m map[string]int  // nil map, can't add to it yet

// Make a map
m = make(map[string]int)
m["age"] = 30
m["score"] = 100

// Map literal
person := map[string]string{
    "name":  "Alice",
    "email": "alice@example.com",
    "city":  "NYC",
}

// Access values
name := person["name"]
age := m["age"]

// Check if key exists
email, exists := person["email"]
if exists {
    fmt.Println("Email:", email)
}

// Delete a key
delete(person, "city")

// Iterate over map
for key, value := range person {
    fmt.Println(key, ":", value)
}

// Get length
fmt.Println(len(person))
```

---

## 1.7 Structs (Custom Types)

Structs are Go's way of creating custom data types (like classes without methods).

```go
// Define a struct
type User struct {
    ID       int
    Username string
    Email    string
    Age      int
    IsActive bool
}

// Create struct instances
func main() {
    // Method 1: Field by field
    var user1 User
    user1.ID = 1
    user1.Username = "alice"
    user1.Email = "alice@example.com"
    user1.Age = 30
    user1.IsActive = true
    
    // Method 2: Struct literal with field names
    user2 := User{
        ID:       2,
        Username: "bob",
        Email:    "bob@example.com",
        Age:      25,
        IsActive: true,
    }
    
    // Method 3: Struct literal without field names (must be in order)
    user3 := User{3, "charlie", "charlie@example.com", 35, false}
    
    // Access fields
    fmt.Println(user1.Username)
    fmt.Println(user2.Email)
    
    // Update fields
    user1.Age = 31
}
```

### Anonymous Structs

```go
// Useful for one-time use
person := struct {
    Name string
    Age  int
}{
    Name: "Alice",
    Age:  30,
}
```

### Struct Tags (Important for JSON!)

```go
type User struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
    Password string `json:"-"`  // Ignore in JSON
    Age      int    `json:"age,omitempty"`  // Omit if zero value
}
```

### Methods on Structs

```go
type User struct {
    Username string
    Age      int
}

// Method with value receiver (doesn't modify original)
func (u User) Greet() string {
    return "Hello, I'm " + u.Username
}

// Method with pointer receiver (can modify original)
func (u *User) HaveBirthday() {
    u.Age++
}

func main() {
    user := User{Username: "Alice", Age: 30}
    
    fmt.Println(user.Greet())  // "Hello, I'm Alice"
    
    user.HaveBirthday()
    fmt.Println(user.Age)  // 31
}
```

---

## 1.8 Pointers

Pointers hold memory addresses. Go has pointers but no pointer arithmetic (safer than C/C++).

```go
func main() {
    x := 42
    
    // & gets the address
    p := &x
    fmt.Println(p)   // Address like: 0xc0000140a0
    
    // * dereferences (gets the value at address)
    fmt.Println(*p)  // 42
    
    // Modify through pointer
    *p = 100
    fmt.Println(x)   // 100 (x changed!)
}
```

### Why Use Pointers?

1. **Modify original values**:
```go
func increment(n *int) {
    *n++
}

func main() {
    num := 5
    increment(&num)
    fmt.Println(num)  // 6
}
```

2. **Efficiency with large structs**:
```go
type LargeStruct struct {
    Data [1000000]int
}

// Passing by pointer is efficient
func process(ls *LargeStruct) {
    // Work with ls
}

func main() {
    ls := LargeStruct{}
    process(&ls)  // Only passes address, not entire struct
}
```

3. **Representing "no value" (nil)**:
```go
var p *int  // p is nil
if p == nil {
    fmt.Println("p is nil")
}
```

---

## 1.9 Error Handling

Go doesn't have exceptions. Errors are returned as values.

```go
import (
    "errors"
    "fmt"
)

// Function that can fail
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// Or use fmt.Errorf for formatted errors
func getUserByID(id int) (*User, error) {
    if id < 0 {
        return nil, fmt.Errorf("invalid user ID: %d", id)
    }
    // ... fetch user logic
    return &User{ID: id}, nil
}

func main() {
    // Always check errors!
    result, err := divide(10, 2)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Result:", result)
    
    // Multiple return values with error
    user, err := getUserByID(123)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("User:", user)
}
```

### Error Handling Patterns

```go
// Pattern 1: Early return
func processUser(id int) error {
    user, err := getUser(id)
    if err != nil {
        return err  // Propagate error up
    }
    
    if err := validateUser(user); err != nil {
        return err
    }
    
    if err := saveUser(user); err != nil {
        return err
    }
    
    return nil
}

// Pattern 2: Wrap errors for context
import "fmt"

func readConfig() error {
    data, err := readFile("config.json")
    if err != nil {
        return fmt.Errorf("failed to read config: %w", err)
    }
    // ...
    return nil
}
```

---

## 1.10 Packages and Imports

### Standard Library Packages

```go
import (
    "fmt"           // Formatted I/O
    "strings"       // String manipulation
    "time"          // Time and dates
    "math"          // Mathematical functions
    "os"            // Operating system functions
    "io"            // I/O interfaces
    "encoding/json" // JSON encoding/decoding
    "net/http"      // HTTP client and server
)
```

### Creating Your Own Package

**File structure**:
```
myapp/
├── main.go
└── utils/
    └── helpers.go
```

**utils/helpers.go**:
```go
package utils

// Exported function (starts with capital letter)
func Capitalize(s string) string {
    if len(s) == 0 {
        return s
    }
    return strings.ToUpper(s[:1]) + s[1:]
}

// Unexported function (starts with lowercase)
func internal() string {
    return "private"
}
```

**main.go**:
```go
package main

import (
    "fmt"
    "myapp/utils"
)

func main() {
    result := utils.Capitalize("hello")
    fmt.Println(result)  // "Hello"
    
    // utils.internal() // Error! Can't access unexported function
}
```

### Module Management

```bash
# Initialize a module
go mod init myapp

# Add dependencies
go get github.com/gorilla/mux

# Remove unused dependencies
go mod tidy
```

---

## 1.11 Working with JSON (Essential for APIs!)

### Encoding (Go → JSON)

```go
import (
    "encoding/json"
    "fmt"
)

type User struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
    Password string `json:"-"`  // Omit from JSON
}

func main() {
    user := User{
        ID:       1,
        Username: "alice",
        Email:    "alice@example.com",
        Password: "secret",
    }
    
    // Convert to JSON
    jsonData, err := json.Marshal(user)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    
    fmt.Println(string(jsonData))
    // Output: {"id":1,"username":"alice","email":"alice@example.com"}
    
    // Pretty print JSON
    prettyJSON, _ := json.MarshalIndent(user, "", "  ")
    fmt.Println(string(prettyJSON))
}
```

### Decoding (JSON → Go)

```go
func main() {
    jsonStr := `{"id":1,"username":"alice","email":"alice@example.com"}`
    
    var user User
    err := json.Unmarshal([]byte(jsonStr), &user)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    
    fmt.Println(user.Username)  // "alice"
    fmt.Println(user.Email)     // "alice@example.com"
}
```

### Working with Dynamic JSON

```go
// When structure is unknown
var data map[string]interface{}
json.Unmarshal([]byte(jsonStr), &data)

name := data["username"].(string)  // Type assertion
```

---

## 1.12 Defer Statement

`defer` schedules a function call to run after the current function returns. Commonly used for cleanup.

```go
func main() {
    file, err := os.Open("data.txt")
    if err != nil {
        log.Fatal(err)
    }
    defer file.Close()  // Will run when main() exits
    
    // Work with file...
    // No matter how function exits, file.Close() will be called
}
```

### Multiple Defers (LIFO order)

```go
func main() {
    defer fmt.Println("First")
    defer fmt.Println("Second")
    defer fmt.Println("Third")
    
    fmt.Println("Main body")
}
// Output:
// Main body
// Third
// Second
// First
```

---

## 1.13 Interfaces

Interfaces define behavior. A type implements an interface by implementing its methods (implicit, no "implements" keyword).

```go
// Define an interface
type Shape interface {
    Area() float64
    Perimeter() float64
}

// Rectangle implements Shape
type Rectangle struct {
    Width, Height float64
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}

// Circle implements Shape
type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return 3.14159 * c.Radius * c.Radius
}

func (c Circle) Perimeter() float64 {
    return 2 * 3.14159 * c.Radius
}

// Function that works with any Shape
func printShapeInfo(s Shape) {
    fmt.Printf("Area: %.2f, Perimeter: %.2f\n", s.Area(), s.Perimeter())
}

func main() {
    rect := Rectangle{Width: 10, Height: 5}
    circle := Circle{Radius: 7}
    
    printShapeInfo(rect)
    printShapeInfo(circle)
}
```

### Empty Interface

```go
// interface{} can hold any type
var anything interface{}

anything = 42
anything = "hello"
anything = []int{1, 2, 3}

// Type assertion
value := anything.(string)  // Panics if wrong type

// Safe type assertion
value, ok := anything.(string)
if ok {
    fmt.Println(value)
}
```

---

## 1.14 Goroutines and Concurrency (Brief Introduction)

Goroutines are lightweight threads managed by Go runtime.

```go
import (
    "fmt"
    "time"
)

func sayHello() {
    fmt.Println("Hello from goroutine!")
}

func main() {
    // Launch goroutine
    go sayHello()
    
    // Launch anonymous function
    go func() {
        fmt.Println("Hello from anonymous goroutine!")
    }()
    
    // Wait for goroutines (simple approach)
    time.Sleep(100 * time.Millisecond)
    
    fmt.Println("Main function")
}
```

**Note**: We'll cover goroutines properly when we need concurrency in our API (rate limiting, background tasks, etc.).

---

## 1.15 Go Commands Cheat Sheet

```bash
# Run Go file
go run main.go

# Build executable
go build main.go

# Format code (Go has official formatting!)
go fmt main.go
go fmt ./...  # Format entire project

# Get dependencies
go get github.com/some/package

# Initialize module
go mod init myapp

# Download dependencies
go mod download

# Clean up dependencies
go mod tidy

# Run tests
go test

# Show documentation
go doc fmt.Println

# Install tools
go install github.com/some/tool@latest
```

---

## Key Takeaways

1. **No Semicolons**: Go adds them automatically
2. **Error Handling**: Always return and check errors (no exceptions)
3. **Multiple Returns**: Functions commonly return (value, error)
4. **Slices over Arrays**: Use slices for dynamic collections
5. **Structs not Classes**: Go uses composition over inheritance
6. **Pointers**: Use for efficiency and mutation
7. **Capital = Exported**: Capital first letter means public
8. **JSON Tags**: Control JSON encoding with struct tags
9. **Defer**: Cleanup with defer
10. **Interfaces**: Implicit implementation

---

## Common Patterns You'll See

### Error Checking Pattern
```go
result, err := someFunction()
if err != nil {
    return err  // or handle error
}
// use result
```

### OK Pattern (map, type assertion)
```go
value, ok := myMap["key"]
if !ok {
    // key doesn't exist
}
```

### Blank Identifier
```go
// Ignore values you don't need
_, err := someFunction()  // Ignore first return value
for _, value := range slice {  // Ignore index
    // use value
}
```

---

## Next Steps

Now that you know Go basics, we'll use this knowledge to build HTTP servers and APIs in the following units. The patterns you've learned here will be used extensively:

- Structs → Request/Response models
- JSON → API data format
- Error handling → API error responses
- Interfaces → Middleware and handlers
- Pointers → Efficient data passing
- Maps → Request headers, query parameters

You're ready to start building APIs!
