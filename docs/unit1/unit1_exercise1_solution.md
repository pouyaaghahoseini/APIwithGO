# Exercise 1 Solution: Student Grade Manager

**Complete implementation with explanations**

---

## Full Solution Code

```go
package main

import (
    "errors"
    "fmt"
)

type Student struct {
    ID        int
    FirstName string
    LastName  string
    Email     string
    Grades    []float64
}

// addStudent creates a new student and adds it to the slice
func addStudent(students []Student, id int, firstName, lastName, email string) []Student {
    newStudent := Student{
        ID:        id,
        FirstName: firstName,
        LastName:  lastName,
        Email:     email,
        Grades:    []float64{}, // Initialize empty slice for grades
    }
    return append(students, newStudent)
}

// addGrade adds a grade to a specific student
func addGrade(students []Student, studentID int, grade float64) error {
    // Validate grade range
    if grade < 0 || grade > 100 {
        return fmt.Errorf("grade must be between 0 and 100, got %.2f", grade)
    }
    
    // Find the student by ID
    for i := range students {
        if students[i].ID == studentID {
            // Found the student, append the grade
            students[i].Grades = append(students[i].Grades, grade)
            return nil
        }
    }
    
    // Student not found
    return fmt.Errorf("student with ID %d not found", studentID)
}

// calculateAverage computes the average grade for a student
func calculateAverage(student Student) float64 {
    // Handle case where student has no grades
    if len(student.Grades) == 0 {
        return 0.0
    }
    
    // Sum all grades
    sum := 0.0
    for _, grade := range student.Grades {
        sum += grade
    }
    
    // Calculate and return average
    return sum / float64(len(student.Grades))
}

// findTopStudent returns the student with the highest average grade
func findTopStudent(students []Student) (*Student, error) {
    // Check if there are any students
    if len(students) == 0 {
        return nil, errors.New("no students in the system")
    }
    
    // Initialize with first student
    topStudent := &students[0]
    highestAverage := calculateAverage(students[0])
    
    // Compare with remaining students
    for i := 1; i < len(students); i++ {
        avg := calculateAverage(students[i])
        if avg > highestAverage {
            highestAverage = avg
            topStudent = &students[i]
        }
    }
    
    return topStudent, nil
}

// displayAllStudents prints information for all students
func displayAllStudents(students []Student) {
    for _, student := range students {
        fmt.Println("--------------------------------------------------")
        fmt.Printf("ID: %d\n", student.ID)
        fmt.Printf("Name: %s %s\n", student.FirstName, student.LastName)
        fmt.Printf("Email: %s\n", student.Email)
        fmt.Printf("Grades: %v\n", student.Grades)
        fmt.Printf("Average: %.2f\n", calculateAverage(student))
    }
    fmt.Println("--------------------------------------------------")
}

func main() {
    fmt.Println("=== Student Grade Management System ===\n")
    
    students := []Student{}
    
    // Add students
    fmt.Println("Adding students...")
    students = addStudent(students, 1, "John", "Doe", "john@example.com")
    fmt.Println("✓ Added: John Doe (john@example.com)")
    
    students = addStudent(students, 2, "Jane", "Smith", "jane@example.com")
    fmt.Println("✓ Added: Jane Smith (jane@example.com)")
    
    students = addStudent(students, 3, "Bob", "Johnson", "bob@example.com")
    fmt.Println("✓ Added: Bob Johnson (bob@example.com)")
    
    // Add grades
    fmt.Println("\nAdding grades...")
    
    grades := []struct {
        studentID int
        grade     float64
    }{
        {1, 85.5},
        {1, 92.0},
        {1, 78.5},
        {2, 95.0},
        {2, 88.5},
        {2, 91.0},
        {3, 72.0},
        {3, 68.5},
    }
    
    for _, g := range grades {
        err := addGrade(students, g.studentID, g.grade)
        if err != nil {
            fmt.Printf("✗ Error: %v\n", err)
        } else {
            fmt.Printf("✓ Added grade %.1f for student ID %d\n", g.grade, g.studentID)
        }
    }
    
    // Display all students
    fmt.Println("\nAll Students:")
    displayAllStudents(students)
    
    // Find top student
    topStudent, err := findTopStudent(students)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Printf("\nTop Student: %s %s with average %.2f\n",
            topStudent.FirstName, topStudent.LastName, calculateAverage(*topStudent))
    }
    
    // Test error handling
    fmt.Println("\n=== Testing Error Handling ===")
    
    // Test invalid student ID
    err = addGrade(students, 999, 90.0)
    if err != nil {
        fmt.Printf("✓ Correctly caught error: %v\n", err)
    }
    
    // Test invalid grade (too high)
    err = addGrade(students, 1, 105.0)
    if err != nil {
        fmt.Printf("✓ Correctly caught error: %v\n", err)
    }
    
    // Test invalid grade (negative)
    err = addGrade(students, 1, -10.0)
    if err != nil {
        fmt.Printf("✓ Correctly caught error: %v\n", err)
    }
    
    // Test finding top student with empty list
    emptyStudents := []Student{}
    _, err = findTopStudent(emptyStudents)
    if err != nil {
        fmt.Printf("✓ Correctly caught error: %v\n", err)
    }
}
```

---

## Key Concepts Explained

### 1. Working with Slices

```go
// Slices are passed by value, but they reference underlying array
func addStudent(students []Student, ...) []Student {
    // append returns a new slice (may have reallocated)
    return append(students, newStudent)
}
```

**Why return the slice?** Because `append` might allocate a new underlying array if capacity is exceeded. We must capture the returned value.

### 2. Modifying Slice Elements

```go
// Loop with index to modify elements
for i := range students {
    if students[i].ID == studentID {
        students[i].Grades = append(students[i].Grades, grade)
        return nil
    }
}
```

**Why use `i` instead of `student`?** When ranging over a slice, the value is a copy. We need to modify the actual element in the slice, so we use the index.

### 3. Error Handling Patterns

```go
// Return error if validation fails
if grade < 0 || grade > 100 {
    return fmt.Errorf("grade must be between 0 and 100, got %.2f", grade)
}

// Check for error before proceeding
err := addGrade(students, 1, 85.5)
if err != nil {
    fmt.Println("Error:", err)
    return
}
```

### 4. Pointer Returns

```go
func findTopStudent(students []Student) (*Student, error) {
    // ...
    topStudent := &students[0]  // Return pointer to element
    return topStudent, nil
}
```

**Why return a pointer?** We're returning a reference to a student that exists in the slice, not a copy.

### 5. Zero Values

```go
if len(student.Grades) == 0 {
    return 0.0  // Return zero value for float64
}
```

Go's zero values make it easy to handle "no data" cases.

---

## Bonus Solutions

### Bonus 1: Grade Statistics

```go
func displayStatistics(students []Student) {
    if len(students) == 0 {
        fmt.Println("No students to analyze")
        return
    }
    
    var allGrades []float64
    totalSum := 0.0
    
    // Collect all grades
    for _, student := range students {
        for _, grade := range student.Grades {
            allGrades = append(allGrades, grade)
            totalSum += grade
        }
    }
    
    if len(allGrades) == 0 {
        fmt.Println("No grades recorded")
        return
    }
    
    // Find highest and lowest
    highest := allGrades[0]
    lowest := allGrades[0]
    
    for _, grade := range allGrades {
        if grade > highest {
            highest = grade
        }
        if grade < lowest {
            lowest = grade
        }
    }
    
    classAverage := totalSum / float64(len(allGrades))
    
    fmt.Println("\n=== Class Statistics ===")
    fmt.Printf("Highest Grade: %.2f\n", highest)
    fmt.Printf("Lowest Grade: %.2f\n", lowest)
    fmt.Printf("Class Average: %.2f\n", classAverage)
    fmt.Printf("Total Grades Recorded: %d\n", len(allGrades))
}
```

### Bonus 2: Find Student by Email

```go
func findStudentByEmail(students []Student, email string) (*Student, error) {
    for i := range students {
        if students[i].Email == email {
            return &students[i], nil
        }
    }
    return nil, fmt.Errorf("student with email %s not found", email)
}

// Usage
student, err := findStudentByEmail(students, "jane@example.com")
if err != nil {
    fmt.Println("Error:", err)
} else {
    fmt.Printf("Found: %s %s\n", student.FirstName, student.LastName)
}
```

### Bonus 3: Remove Student

```go
func removeStudent(students []Student, studentID int) ([]Student, error) {
    // Find student index
    index := -1
    for i := range students {
        if students[i].ID == studentID {
            index = i
            break
        }
    }
    
    if index == -1 {
        return students, fmt.Errorf("student with ID %d not found", studentID)
    }
    
    // Create new slice without the student
    // Method 1: Using append to concatenate slices before and after
    newStudents := append(students[:index], students[index+1:]...)
    
    return newStudents, nil
}

// Alternative method using manual copy
func removeStudentAlt(students []Student, studentID int) ([]Student, error) {
    newStudents := []Student{}
    found := false
    
    for _, student := range students {
        if student.ID != studentID {
            newStudents = append(newStudents, student)
        } else {
            found = true
        }
    }
    
    if !found {
        return students, fmt.Errorf("student with ID %d not found", studentID)
    }
    
    return newStudents, nil
}
```

### Bonus 4: Grade Letter

```go
func (s Student) GetLetterGrade() string {
    avg := calculateAverage(s)
    
    switch {
    case avg >= 90:
        return "A"
    case avg >= 80:
        return "B"
    case avg >= 70:
        return "C"
    case avg >= 60:
        return "D"
    default:
        return "F"
    }
}

// Usage in displayAllStudents
func displayAllStudents(students []Student) {
    for _, student := range students {
        fmt.Println("--------------------------------------------------")
        fmt.Printf("ID: %d\n", student.ID)
        fmt.Printf("Name: %s %s\n", student.FirstName, student.LastName)
        fmt.Printf("Email: %s\n", student.Email)
        fmt.Printf("Grades: %v\n", student.Grades)
        fmt.Printf("Average: %.2f (%s)\n", calculateAverage(student), student.GetLetterGrade())
    }
    fmt.Println("--------------------------------------------------")
}
```

---

## Common Mistakes and How to Avoid Them

### Mistake 1: Not Capturing append Result

```go
// WRONG
func addStudent(students []Student, ...) {
    append(students, newStudent)  // Lost!
}

// RIGHT
func addStudent(students []Student, ...) []Student {
    return append(students, newStudent)
}
```

### Mistake 2: Modifying Copy in Range Loop

```go
// WRONG - modifies copy, not original
for _, student := range students {
    student.Grades = append(student.Grades, grade)  // Doesn't affect original
}

// RIGHT - use index
for i := range students {
    students[i].Grades = append(students[i].Grades, grade)
}
```

### Mistake 3: Not Checking Errors

```go
// WRONG
addGrade(students, 1, 85.5)  // Ignoring error

// RIGHT
err := addGrade(students, 1, 85.5)
if err != nil {
    fmt.Println("Error:", err)
}
```

### Mistake 4: Division by Zero

```go
// WRONG
func calculateAverage(student Student) float64 {
    sum := 0.0
    for _, grade := range student.Grades {
        sum += grade
    }
    return sum / float64(len(student.Grades))  // Panic if len == 0!
}

// RIGHT
func calculateAverage(student Student) float64 {
    if len(student.Grades) == 0 {
        return 0.0
    }
    // ... rest of calculation
}
```

---

## Testing Output

When you run the complete solution, you should see:

```
=== Student Grade Management System ===

Adding students...
✓ Added: John Doe (john@example.com)
✓ Added: Jane Smith (jane@example.com)
✓ Added: Bob Johnson (bob@example.com)

Adding grades...
✓ Added grade 85.5 for student ID 1
✓ Added grade 92.0 for student ID 1
✓ Added grade 78.5 for student ID 1
✓ Added grade 95.0 for student ID 2
✓ Added grade 88.5 for student ID 2
✓ Added grade 91.0 for student ID 2
✓ Added grade 72.0 for student ID 3
✓ Added grade 68.5 for student ID 3

All Students:
--------------------------------------------------
ID: 1
Name: John Doe
Email: john@example.com
Grades: [85.5 92 78.5]
Average: 85.33
--------------------------------------------------
ID: 2
Name: Jane Smith
Email: jane@example.com
Grades: [95 88.5 91]
Average: 91.50
--------------------------------------------------
ID: 3
Name: Bob Johnson
Email: bob@example.com
Grades: [72 68.5]
Average: 70.25
--------------------------------------------------

Top Student: Jane Smith with average 91.50

=== Testing Error Handling ===
✓ Correctly caught error: student with ID 999 not found
✓ Correctly caught error: grade must be between 0 and 100, got 105.00
✓ Correctly caught error: grade must be between 0 and 100, got -10.00
✓ Correctly caught error: no students in the system
```

---

## What You've Learned

✅ Creating and working with structs  
✅ Managing slices (append, iterate, modify)  
✅ Writing functions with multiple return values  
✅ Proper error handling patterns  
✅ Using pointers when appropriate  
✅ Range loops with index vs value  
✅ Validation and edge case handling  
✅ Methods on structs  

These skills are foundational for API development where you'll work with user data, validation, and error responses!
