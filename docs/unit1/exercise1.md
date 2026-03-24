# Exercise 1: Student Grade Manager

**Difficulty**: Beginner  
**Estimated Time**: 30-45 minutes  
**Concepts Covered**: Structs, slices, functions, loops, error handling, maps

---

## Objective

Build a command-line student grade management system that allows you to:
1. Add students
2. Record grades for students
3. Calculate average grade for a student
4. Find the top-performing student
5. Display all students and their information

---

## Requirements

### Data Structures

Create the following structs:

```go
type Student struct {
    ID        int
    FirstName string
    LastName  string
    Email     string
    Grades    []float64  // Slice to store multiple grades
}
```

### Functions to Implement

#### 1. `addStudent`

`addStudent(students []Student, id int, firstName, lastName, email string) []Student`

Add a new student to the slice.

#### 2. `addGrade`

`addGrade(students []Student, studentID int, grade float64) error`

Add a grade to a specific student. Return an error if student ID is not found. Return an error if grade is not between 0 and 100.

#### 3. `calculateAverage`

`calculateAverage(student Student) float64`

Calculate and return the average of all grades. Return 0.0 if student has no grades.

#### 4. `findTopStudent`

`findTopStudent(students []Student) (*Student, error)`

Find and return the student with the highest average. Return an error if no students exist.

#### 5. `displayAllStudents`

`displayAllStudents(students []Student)`

Print all students with their information and average grade.

---

## Example Output

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
```

---

## Starter Code

```go
package main

import (
    "fmt"
    "errors"
)

type Student struct {
    ID        int
    FirstName string
    LastName  string
    Email     string
    Grades    []float64
}

// TODO: Implement addStudent
func addStudent(students []Student, id int, firstName, lastName, email string) []Student {
    // Your code here
}

// TODO: Implement addGrade
func addGrade(students []Student, studentID int, grade float64) error {
    // Your code here
}

// TODO: Implement calculateAverage
func calculateAverage(student Student) float64 {
    // Your code here
}

// TODO: Implement findTopStudent
func findTopStudent(students []Student) (*Student, error) {
    // Your code here
}

// TODO: Implement displayAllStudents
func displayAllStudents(students []Student) {
    // Your code here
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
    addGrade(students, 1, 85.5)
    addGrade(students, 1, 92.0)
    addGrade(students, 1, 78.5)
    addGrade(students, 2, 95.0)
    addGrade(students, 2, 88.5)
    addGrade(students, 2, 91.0)
    addGrade(students, 3, 72.0)
    addGrade(students, 3, 68.5)
    
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
}
```

---

## Hints

### Hint 1: addStudent
```go
func addStudent(students []Student, id int, firstName, lastName, email string) []Student {
    newStudent := Student{
        ID:        id,
        FirstName: firstName,
        LastName:  lastName,
        Email:     email,
        Grades:    []float64{},  // Initialize empty slice
    }
    return append(students, newStudent)
}
```

### Hint 2: Finding a student by ID
```go
// Loop through students
for i := range students {
    if students[i].ID == studentID {
        // Found the student at index i
        // Remember: students is a slice of values, so use &students[i] for pointer
    }
}
```

### Hint 3: Calculating average
```go
// Check if there are any grades
if len(student.Grades) == 0 {
    return 0.0
}

// Sum all grades
sum := 0.0
for _, grade := range student.Grades {
    sum += grade
}

// Calculate average
return sum / float64(len(student.Grades))
```

---

## Bonus Challenges

If you finish early, try these additional features:

### Bonus 1: Grade Statistics
Add a function that calculates and displays:
- Highest grade across all students
- Lowest grade across all students
- Overall class average

```go
func displayStatistics(students []Student) {
    // Your code here
}
```

### Bonus 2: Find Student by Email
```go
func findStudentByEmail(students []Student, email string) (*Student, error) {
    // Your code here
}
```

### Bonus 3: Remove Student
```go
func removeStudent(students []Student, studentID int) ([]Student, error) {
    // Your code here
    // Hint: Create a new slice without the student to remove
}
```

### Bonus 4: Grade Letter
Add a method to Student that returns their letter grade:
```go
func (s Student) GetLetterGrade() string {
    avg := calculateAverage(s)
    // Return "A", "B", "C", "D", or "F"
}
```

---

## Testing Your Code

Test with these cases:

1. **Normal operations**: Add students and grades as shown
2. **Error cases**: 
   - Try adding a grade with invalid student ID
   - Try adding a grade of 105 (should fail)
   - Try adding a grade of -10 (should fail)
3. **Edge cases**:
   - Try finding top student when no students exist
   - Try calculating average for student with no grades

---

## What You're Learning

- ✓ Creating and using structs
- ✓ Working with slices (append, iterate)
- ✓ Function parameter passing (value vs pointer)
- ✓ Error handling and returning errors
- ✓ Loops and range
- ✓ Conditionals
- ✓ String formatting with fmt

This exercise prepares you for working with data structures in APIs (users, posts, etc.) and proper error handling!
