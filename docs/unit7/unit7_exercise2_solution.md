# Unit 7 - Exercise 2 Solution: Cursor-Based Pagination for Infinite Scroll

**Complete implementation with explanations**

---

## Full Solution Code

```go
package main

import (
    "encoding/base64"
    "encoding/json"
    "fmt"
    "net/http"
    "sort"
    "strconv"
    "sync"
    "time"

    "github.com/gorilla/mux"
)

// =============================================================================
// MODELS
// =============================================================================

type Post struct {
    ID        int       `json:"id"`
    Title     string    `json:"title"`
    Content   string    `json:"content"`
    Author    string    `json:"author"`
    ViewCount int       `json:"view_count"`
    CreatedAt time.Time `json:"created_at"`
}

type Comment struct {
    ID        int       `json:"id"`
    PostID    int       `json:"post_id"`
    Author    string    `json:"author"`
    Content   string    `json:"content"`
    CreatedAt time.Time `json:"created_at"`
}

// =============================================================================
// CURSOR TYPES
// =============================================================================

type IDCursor struct {
    ID int `json:"id"`
}

type TimeCursor struct {
    CreatedAt time.Time `json:"created_at"`
    ID        int       `json:"id"` // Tiebreaker for same timestamp
}

// =============================================================================
// RESPONSE TYPES
// =============================================================================

type CursorPaginatedResponse struct {
    Data        interface{} `json:"data"`
    Cursors     Cursors     `json:"cursors"`
    HasMore     bool        `json:"has_more"`
    HasPrevious bool        `json:"has_previous,omitempty"`
}

type Cursors struct {
    Next     string `json:"next,omitempty"`
    Previous string `json:"previous,omitempty"`
}

// =============================================================================
// STORAGE
// =============================================================================

var (
    posts      = make(map[int]Post)
    comments   = make(map[int][]Comment)
    nextPostID = 1
    nextCommID = 1
    storageMu  sync.RWMutex
)

// =============================================================================
// CURSOR ENCODING/DECODING
// =============================================================================

func encodeCursor(cursor interface{}) string {
    data, err := json.Marshal(cursor)
    if err != nil {
        return ""
    }
    return base64.StdEncoding.EncodeToString(data)
}

func decodeIDCursor(cursorStr string) (IDCursor, error) {
    if cursorStr == "" {
        return IDCursor{}, nil
    }

    data, err := base64.StdEncoding.DecodeString(cursorStr)
    if err != nil {
        return IDCursor{}, fmt.Errorf("invalid cursor format")
    }

    var cursor IDCursor
    err = json.Unmarshal(data, &cursor)
    if err != nil {
        return IDCursor{}, fmt.Errorf("invalid cursor data")
    }

    return cursor, nil
}

func decodeTimeCursor(cursorStr string) (TimeCursor, error) {
    if cursorStr == "" {
        return TimeCursor{}, nil
    }

    data, err := base64.StdEncoding.DecodeString(cursorStr)
    if err != nil {
        return TimeCursor{}, fmt.Errorf("invalid cursor format")
    }

    var cursor TimeCursor
    err = json.Unmarshal(data, &cursor)
    if err != nil {
        return TimeCursor{}, fmt.Errorf("invalid cursor data")
    }

    return cursor, nil
}

// =============================================================================
// QUERY FUNCTIONS
// =============================================================================

func queryPostsAfter(afterID, limit int) []Post {
    storageMu.RLock()
    defer storageMu.RUnlock()

    // Collect posts with ID > afterID
    candidates := []Post{}
    for _, post := range posts {
        if post.ID > afterID {
            candidates = append(candidates, post)
        }
    }

    // Sort by ID ascending
    sort.Slice(candidates, func(i, j int) bool {
        return candidates[i].ID < candidates[j].ID
    })

    // Take first `limit` items
    if len(candidates) > limit {
        candidates = candidates[:limit]
    }

    return candidates
}

func queryPostsBefore(beforeTime time.Time, beforeID, limit int) []Post {
    storageMu.RLock()
    defer storageMu.RUnlock()

    // Collect posts before the cursor time
    candidates := []Post{}
    for _, post := range posts {
        // Posts created before cursor time
        if post.CreatedAt.Before(beforeTime) {
            candidates = append(candidates, post)
        } else if post.CreatedAt.Equal(beforeTime) && post.ID < beforeID {
            // Same timestamp but lower ID (tiebreaker)
            candidates = append(candidates, post)
        }
    }

    // Sort by CreatedAt DESC, then ID DESC
    sort.Slice(candidates, func(i, j int) bool {
        if candidates[i].CreatedAt.Equal(candidates[j].CreatedAt) {
            return candidates[i].ID > candidates[j].ID
        }
        return candidates[i].CreatedAt.After(candidates[j].CreatedAt)
    })

    // Take first `limit` items
    if len(candidates) > limit {
        candidates = candidates[:limit]
    }

    return candidates
}

func queryCommentsAfter(postID, afterID, limit int) []Comment {
    storageMu.RLock()
    defer storageMu.RUnlock()

    postComments, exists := comments[postID]
    if !exists {
        return []Comment{}
    }

    // Filter comments after cursor
    candidates := []Comment{}
    for _, comment := range postComments {
        if comment.ID > afterID {
            candidates = append(candidates, comment)
        }
    }

    // Sort by ID ascending (chronological)
    sort.Slice(candidates, func(i, j int) bool {
        return candidates[i].ID < candidates[j].ID
    })

    // Take first `limit` items
    if len(candidates) > limit {
        candidates = candidates[:limit]
    }

    return candidates
}

// =============================================================================
// HANDLERS
// =============================================================================

func getPosts(w http.ResponseWriter, r *http.Request) {
    // Parse parameters
    cursorStr := r.URL.Query().Get("cursor")
    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))

    if limit < 1 {
        limit = 20
    }
    if limit > 100 {
        limit = 100
    }

    // Decode cursor
    cursor, err := decodeIDCursor(cursorStr)
    if err != nil {
        respondError(w, http.StatusBadRequest, "Invalid cursor")
        return
    }

    // Query posts after cursor (fetch limit+1 to check has_more)
    posts := queryPostsAfter(cursor.ID, limit+1)

    // Check if there are more results
    hasMore := len(posts) > limit
    if hasMore {
        posts = posts[:limit] // Trim to actual limit
    }

    // Generate next cursor from last item
    var nextCursor string
    if hasMore && len(posts) > 0 {
        lastPost := posts[len(posts)-1]
        nextCursor = encodeCursor(IDCursor{ID: lastPost.ID})
    }

    // Build response
    response := CursorPaginatedResponse{
        Data: posts,
        Cursors: Cursors{
            Next: nextCursor,
        },
        HasMore: hasMore,
    }

    respondJSON(w, http.StatusOK, response)
}

func getPostsFeed(w http.ResponseWriter, r *http.Request) {
    // Parse parameters
    cursorStr := r.URL.Query().Get("cursor")
    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))

    if limit < 1 {
        limit = 10
    }
    if limit > 50 {
        limit = 50
    }

    // Decode time-based cursor
    var beforeTime time.Time
    var beforeID int

    if cursorStr != "" {
        cursor, err := decodeTimeCursor(cursorStr)
        if err != nil {
            respondError(w, http.StatusBadRequest, "Invalid cursor")
            return
        }
        beforeTime = cursor.CreatedAt
        beforeID = cursor.ID
    } else {
        // No cursor = start from now
        beforeTime = time.Now().Add(time.Hour) // Future time
        beforeID = 999999
    }

    // Query posts before cursor (fetch limit+1 to check has_more)
    posts := queryPostsBefore(beforeTime, beforeID, limit+1)

    // Check if there are more results
    hasMore := len(posts) > limit
    if hasMore {
        posts = posts[:limit]
    }

    // Generate next cursor from last item
    var nextCursor string
    if hasMore && len(posts) > 0 {
        lastPost := posts[len(posts)-1]
        nextCursor = encodeCursor(TimeCursor{
            CreatedAt: lastPost.CreatedAt,
            ID:        lastPost.ID,
        })
    }

    // Build response
    response := CursorPaginatedResponse{
        Data: posts,
        Cursors: Cursors{
            Next: nextCursor,
        },
        HasMore: hasMore,
    }

    respondJSON(w, http.StatusOK, response)
}

func createPost(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Title   string `json:"title"`
        Content string `json:"content"`
        Author  string `json:"author"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Validate
    if req.Title == "" || req.Content == "" {
        respondError(w, http.StatusBadRequest, "Title and content are required")
        return
    }

    // Create post
    storageMu.Lock()
    post := Post{
        ID:        nextPostID,
        Title:     req.Title,
        Content:   req.Content,
        Author:    req.Author,
        ViewCount: 0,
        CreatedAt: time.Now(),
    }
    posts[nextPostID] = post
    nextPostID++
    storageMu.Unlock()

    respondJSON(w, http.StatusCreated, post)
}

func getComments(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    postID, _ := strconv.Atoi(vars["post_id"])

    // Parse parameters
    cursorStr := r.URL.Query().Get("cursor")
    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))

    if limit < 1 {
        limit = 20
    }
    if limit > 100 {
        limit = 100
    }

    // Decode cursor
    cursor, err := decodeIDCursor(cursorStr)
    if err != nil {
        respondError(w, http.StatusBadRequest, "Invalid cursor")
        return
    }

    // Query comments after cursor
    comments := queryCommentsAfter(postID, cursor.ID, limit+1)

    // Check has_more
    hasMore := len(comments) > limit
    if hasMore {
        comments = comments[:limit]
    }

    // Generate next cursor
    var nextCursor string
    if hasMore && len(comments) > 0 {
        lastComment := comments[len(comments)-1]
        nextCursor = encodeCursor(IDCursor{ID: lastComment.ID})
    }

    // Build response
    response := CursorPaginatedResponse{
        Data: comments,
        Cursors: Cursors{
            Next: nextCursor,
        },
        HasMore: hasMore,
    }

    respondJSON(w, http.StatusOK, response)
}

func createComment(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    postID, _ := strconv.Atoi(vars["post_id"])

    var req struct {
        Author  string `json:"author"`
        Content string `json:"content"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Validate
    if req.Content == "" {
        respondError(w, http.StatusBadRequest, "Content is required")
        return
    }

    // Check post exists
    storageMu.RLock()
    _, exists := posts[postID]
    storageMu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    // Create comment
    storageMu.Lock()
    comment := Comment{
        ID:        nextCommID,
        PostID:    postID,
        Author:    req.Author,
        Content:   req.Content,
        CreatedAt: time.Now(),
    }
    comments[postID] = append(comments[postID], comment)
    nextCommID++
    storageMu.Unlock()

    respondJSON(w, http.StatusCreated, comment)
}

// =============================================================================
// HELPERS
// =============================================================================

func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}

func seedDatabase() {
    storageMu.Lock()
    defer storageMu.Unlock()

    now := time.Now()

    // Create 200 posts
    for i := 1; i <= 200; i++ {
        post := Post{
            ID:        i,
            Title:     fmt.Sprintf("Post %d", i),
            Content:   fmt.Sprintf("Content for post %d", i),
            Author:    fmt.Sprintf("author%d", i%10),
            ViewCount: i * 10,
            CreatedAt: now.Add(-time.Duration(200-i) * time.Hour),
        }
        posts[i] = post

        // Add some comments
        for j := 0; j < (i % 5); j++ {
            comment := Comment{
                ID:        nextCommID,
                PostID:    i,
                Author:    fmt.Sprintf("commenter%d", j),
                Content:   fmt.Sprintf("Comment %d on post %d", j, i),
                CreatedAt: now.Add(-time.Duration(200-i) * time.Hour),
            }
            comments[i] = append(comments[i], comment)
            nextCommID++
        }
    }

    nextPostID = 201
}

// =============================================================================
// MAIN
// =============================================================================

func main() {
    seedDatabase()

    r := mux.NewRouter()

    // Post routes
    r.HandleFunc("/posts", getPosts).Methods("GET")
    r.HandleFunc("/posts/feed", getPostsFeed).Methods("GET")
    r.HandleFunc("/posts", createPost).Methods("POST")

    // Comment routes
    r.HandleFunc("/comments/{post_id}", getComments).Methods("GET")
    r.HandleFunc("/comments/{post_id}", createComment).Methods("POST")

    fmt.Println("Server starting on :8080")
    fmt.Println("\nExample requests:")
    fmt.Println("  ID-based cursor (posts):")
    fmt.Println("    http://localhost:8080/posts?limit=20")
    fmt.Println("\n  Time-based cursor (feed):")
    fmt.Println("    http://localhost:8080/posts/feed?limit=10")
    fmt.Println("\n  Comments:")
    fmt.Println("    http://localhost:8080/comments/10?limit=5")

    http.ListenAndServe(":8080", r)
}
```

---

## Key Concepts Explained

### 1. Cursor Encoding/Decoding

```go
// ID-based cursor
type IDCursor struct {
    ID int `json:"id"`
}

// Encode to base64
func encodeCursor(cursor interface{}) string {
    data, _ := json.Marshal(cursor)  // {"id": 20}
    return base64.StdEncoding.EncodeToString(data)  // "eyJpZCI6MjB9"
}

// Decode from base64
func decodeIDCursor(cursorStr string) (IDCursor, error) {
    data, _ := base64.StdEncoding.DecodeString(cursorStr)
    var cursor IDCursor
    json.Unmarshal(data, &cursor)
    return cursor, nil
}
```

**Why base64?**:
- Makes cursor opaque to clients
- URL-safe encoding
- Prevents cursor manipulation
- Can encode complex data structures

### 2. ID-Based Cursor Pagination

```go
func queryPostsAfter(afterID, limit int) []Post {
    candidates := []Post{}
    
    // Find all posts with ID > cursor
    for _, post := range posts {
        if post.ID > afterID {
            candidates = append(candidates, post)
        }
    }
    
    // Sort by ID ascending
    sort.Slice(candidates, func(i, j int) bool {
        return candidates[i].ID < candidates[j].ID
    })
    
    // Take first `limit` items
    if len(candidates) > limit {
        candidates = candidates[:limit]
    }
    
    return candidates
}
```

**SQL equivalent**:
```sql
SELECT * FROM posts 
WHERE id > ? 
ORDER BY id 
LIMIT ?
```

**Performance**: Uses index on ID (very fast, even for deep pagination)

### 3. Has More Check

```go
// Fetch limit+1 to check if there are more results
posts := queryPostsAfter(afterID, limit+1)

// If we got more than requested, there are more pages
hasMore := len(posts) > limit
if hasMore {
    posts = posts[:limit]  // Trim the extra item
}

// Generate next cursor from last item
var nextCursor string
if hasMore && len(posts) > 0 {
    lastPost := posts[len(posts)-1]
    nextCursor = encodeCursor(IDCursor{ID: lastPost.ID})
}
```

**Why limit+1?**:
- Efficiently checks if more data exists
- Avoids separate COUNT query
- Only fetches one extra record

### 4. Time-Based Cursor

```go
type TimeCursor struct {
    CreatedAt time.Time `json:"created_at"`
    ID        int       `json:"id"`  // Tiebreaker for same timestamp
}

func queryPostsBefore(beforeTime time.Time, beforeID, limit int) []Post {
    candidates := []Post{}
    
    for _, post := range posts {
        // Posts created before cursor time
        if post.CreatedAt.Before(beforeTime) {
            candidates = append(candidates, post)
        } else if post.CreatedAt.Equal(beforeTime) && post.ID < beforeID {
            // Same timestamp: use ID as tiebreaker
            candidates = append(candidates, post)
        }
    }
    
    // Sort by time DESC (newest first)
    sort.Slice(candidates, func(i, j int) bool {
        if candidates[i].CreatedAt.Equal(candidates[j].CreatedAt) {
            return candidates[i].ID > candidates[j].ID
        }
        return candidates[i].CreatedAt.After(candidates[j].CreatedAt)
    })
    
    return candidates
}
```

**Why composite cursor?**:
- Multiple posts can have same timestamp
- ID acts as tiebreaker
- Ensures deterministic ordering

### 5. Consistency During Writes

```go
// Scenario: User loads page 1, new post created, user loads page 2

// Page 1 request
cursor1 := ""
posts1 := queryPostsAfter(0, 20)  // Posts 1-20
nextCursor := encodeCursor({ID: 20})

// NEW POST CREATED (ID 201)

// Page 2 request
cursor2 := nextCursor
posts2 := queryPostsAfter(20, 20)  // Posts 21-40 (NOT affected by new post)
```

**Result**: Pagination sequence remains consistent even when data changes.

**Contrast with offset**:
```go
// With offset pagination
page1 := getAllPosts()[0:20]    // Posts 1-20

// NEW POST INSERTED AT BEGINNING

page2 := getAllPosts()[20:40]   // Posts 21-41 (includes post that was at position 20)
// Problem: Post 20 appears in BOTH pages!
```

---

## Request Flow Examples

### Example 1: ID-Based Pagination

**Request 1** (First page):
```bash
GET /posts?limit=20
```

**Processing**:
1. No cursor → afterID = 0
2. Query posts WHERE id > 0 ORDER BY id LIMIT 21
3. Got 21 posts → has_more = true
4. Take first 20 posts
5. Last post ID = 20
6. Generate cursor: `encodeCursor({ID: 20})` → `"eyJpZCI6MjB9"`

**Response**:
```json
{
  "data": [20 posts with IDs 1-20],
  "cursors": {
    "next": "eyJpZCI6MjB9"
  },
  "has_more": true
}
```

**Request 2** (Next page):
```bash
GET /posts?cursor=eyJpZCI6MjB9&limit=20
```

**Processing**:
1. Decode cursor → afterID = 20
2. Query posts WHERE id > 20 ORDER BY id LIMIT 21
3. Got 21 posts → has_more = true
4. Take first 20 posts (IDs 21-40)
5. Generate next cursor: `"eyJpZCI6NDB9"`

**Response**:
```json
{
  "data": [20 posts with IDs 21-40],
  "cursors": {
    "next": "eyJpZCI6NDB9"
  },
  "has_more": true
}
```

### Example 2: Time-Based Feed

**Request 1** (Most recent):
```bash
GET /posts/feed?limit=10
```

**Processing**:
1. No cursor → beforeTime = now + 1 hour (future)
2. Query posts WHERE created_at < beforeTime ORDER BY created_at DESC LIMIT 11
3. Got 11 posts → has_more = true
4. Take first 10 posts (most recent)
5. Last post: created_at = "2024-03-20T10:00:00Z", ID = 190
6. Generate cursor: `encodeCursor({CreatedAt: time, ID: 190})`

**Response**:
```json
{
  "data": [10 most recent posts],
  "cursors": {
    "next": "eyJjcmVhdGVkX2F0IjoiMjAyNC0wMy0yMFQxMDowMDowMFoiLCJpZCI6MTkwfQ=="
  },
  "has_more": true
}
```

---

## Testing the Solution

### Test 1: Basic Cursor Pagination

```bash
# First page
curl "http://localhost:8080/posts?limit=20" | jq '.'

# Save cursor
CURSOR=$(curl -s "http://localhost:8080/posts?limit=20" | jq -r '.cursors.next')

# Next page
curl "http://localhost:8080/posts?cursor=$CURSOR&limit=20" | jq '.'
```

### Test 2: Consistency During Writes

```bash
# Terminal 1: Get first page
curl -s "http://localhost:8080/posts?limit=20" > page1.json
CURSOR=$(jq -r '.cursors.next' page1.json)

# Terminal 2: Create new post
curl -X POST "http://localhost:8080/posts" \
  -H "Content-Type: application/json" \
  -d '{"title":"New Post","content":"New content","author":"alice"}'

# Terminal 1: Get second page
curl -s "http://localhost:8080/posts?cursor=$CURSOR&limit=20" > page2.json

# Check: page2 should have IDs 21-40, NOT affected by new post
jq '.data[0].id' page2.json  # Should be 21
```

**Expected**: New post (ID 201) doesn't appear in the pagination sequence.

### Test 3: Infinite Scroll Simulation

```bash
#!/bin/bash
CURSOR=""
PAGE=1

while true; do
  if [ -z "$CURSOR" ]; then
    RESPONSE=$(curl -s "http://localhost:8080/posts?limit=20")
  else
    RESPONSE=$(curl -s "http://localhost:8080/posts?cursor=$CURSOR&limit=20")
  fi
  
  echo "Page $PAGE:"
  echo $RESPONSE | jq '.data | length'
  
  HAS_MORE=$(echo $RESPONSE | jq -r '.has_more')
  if [ "$HAS_MORE" = "false" ]; then
    echo "Reached end of data"
    break
  fi
  
  CURSOR=$(echo $RESPONSE | jq -r '.cursors.next')
  PAGE=$((PAGE + 1))
  
  if [ $PAGE -gt 15 ]; then
    echo "Stopping after 15 pages"
    break
  fi
done
```

### Test 4: Time-Based Feed

```bash
# Most recent posts
curl "http://localhost:8080/posts/feed?limit=10" | jq '.data[] | {id, created_at}'

# Should be sorted newest first
```

### Test 5: Comment Pagination

```bash
# Get comments for post
curl "http://localhost:8080/comments/10?limit=5" | jq '.'

# Next page of comments
CURSOR=$(curl -s "http://localhost:8080/comments/10?limit=5" | jq -r '.cursors.next')
curl "http://localhost:8080/comments/10?cursor=$CURSOR&limit=5" | jq '.'
```

---

## Performance Comparison

### Offset Pagination (Inefficient for Deep Pages)

```sql
-- Page 10,000 with offset
SELECT * FROM posts 
ORDER BY id 
LIMIT 20 OFFSET 200000

-- Database must:
-- 1. Scan 200,020 rows
-- 2. Discard first 200,000
-- 3. Return 20 rows
-- Time: ~500ms
```

### Cursor Pagination (Fast at Any Depth)

```sql
-- Page 10,000 with cursor
SELECT * FROM posts 
WHERE id > 200000 
ORDER BY id 
LIMIT 20

-- Database uses index:
-- 1. Seek directly to id=200000
-- 2. Read next 20 rows
-- 3. Return 20 rows
-- Time: ~5ms
```

**Performance gain**: 100x faster for deep pagination!

---

## What You've Learned

✅ **Cursor-based pagination** for large datasets  
✅ **Base64 encoding** for opaque cursors  
✅ **ID-based cursors** for simple ordering  
✅ **Time-based cursors** for feeds  
✅ **Composite cursors** with tiebreakers  
✅ **Has more detection** with limit+1  
✅ **Consistency during writes** (new data doesn't affect active pagination)  
✅ **Performance optimization** (WHERE vs OFFSET)  
✅ **Infinite scroll** support  

You now understand cursor pagination used by Twitter, Facebook, Instagram, and other high-scale platforms! 🚀
