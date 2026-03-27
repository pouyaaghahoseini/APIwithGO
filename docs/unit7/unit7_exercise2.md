# Unit 7 - Exercise 2: Cursor-Based Pagination for Infinite Scroll

**Difficulty**: Advanced  
**Estimated Time**: 60-75 minutes  
**Concepts Covered**: Cursor pagination, base64 encoding, infinite scroll, performance optimization, consistency

---

## Objective

Implement cursor-based pagination for a Blog API that supports:
- Efficient pagination for large datasets
- Consistent results during concurrent writes
- Infinite scroll UIs
- Multiple cursor strategies
- Bidirectional pagination (next/previous)

---

## Requirements

### API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | /posts | List posts with cursor pagination |
| GET | /posts/feed | Personalized feed (time-based cursor) |
| POST | /posts | Create new post |
| GET | /comments/{post_id} | List comments for post |
| POST | /comments/{post_id} | Add comment |

### Cursor Types

1. **ID-based cursor**: For consistent ordering
   ```json
   {"cursor": "eyJpZCI6MjB9", "limit": 20}
   ```

2. **Time-based cursor**: For feeds
   ```json
   {"cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNC0wMS0xNVQxMDozMDowMFoifQ==", "limit": 20}
   ```

3. **Composite cursor**: For complex sorting
   ```json
   {"cursor": "eyJpZCI6MjAsImNyZWF0ZWRfYXQiOiIyMDI0LTAxLTE1VDEwOjMwOjAwWiJ9", "limit": 20}
   ```

### Response Format

```json
{
  "data": [...],
  "cursors": {
    "next": "eyJpZCI6NDB9",
    "previous": "eyJpZCI6MjB9"
  },
  "has_more": true,
  "has_previous": false
}
```

---

## Starter Code

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

// Cursor types
type IDCursor struct {
    ID int `json:"id"`
}

type TimeCursor struct {
    CreatedAt time.Time `json:"created_at"`
    ID        int       `json:"id"` // Tiebreaker
}

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

// Storage
var (
    posts      = make(map[int]Post)
    comments   = make(map[int][]Comment)
    nextPostID = 1
    nextCommID = 1
    storageMu  sync.RWMutex
)

// TODO: Implement encodeCursor
func encodeCursor(cursor interface{}) string {
    // Marshal to JSON, then base64 encode
}

// TODO: Implement decodeIDCursor
func decodeIDCursor(cursorStr string) (IDCursor, error) {
    // Base64 decode, then unmarshal to IDCursor
}

// TODO: Implement decodeTimeCursor
func decodeTimeCursor(cursorStr string) (TimeCursor, error) {
    // Base64 decode, then unmarshal to TimeCursor
}

// TODO: Implement getPosts with ID-based cursor
func getPosts(w http.ResponseWriter, r *http.Request) {
    // 1. Parse cursor and limit
    // 2. Decode cursor if present
    // 3. Query posts after cursor
    // 4. Fetch limit+1 to check has_more
    // 5. Build response with next cursor
}

// TODO: Implement getPostsFeed with time-based cursor
func getPostsFeed(w http.ResponseWriter, r *http.Request) {
    // 1. Parse cursor (time-based)
    // 2. Query posts before cursor time
    // 3. Sort by created_at DESC
    // 4. Build response
}

// TODO: Implement createPost
func createPost(w http.ResponseWriter, r *http.Request) {
    // Create new post
    // Demonstrate consistency: new posts don't affect existing pagination
}

// TODO: Implement getComments with cursor
func getComments(w http.ResponseWriter, r *http.Request) {
    // Cursor pagination for comments
}

// TODO: Implement createComment
func createComment(w http.ResponseWriter, r *http.Request) {
    // Add comment to post
}

// TODO: Implement queryPostsAfter
func queryPostsAfter(afterID, limit int) []Post {
    // Query posts WHERE id > afterID ORDER BY id LIMIT limit
}

// TODO: Implement queryPostsBefore (for feed)
func queryPostsBefore(beforeTime time.Time, limit int) []Post {
    // Query posts WHERE created_at < beforeTime ORDER BY created_at DESC LIMIT limit
}

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

func main() {
    seedDatabase()

    r := mux.NewRouter()

    // TODO: Register routes
    // r.HandleFunc("/posts", getPosts).Methods("GET")
    // r.HandleFunc("/posts/feed", getPostsFeed).Methods("GET")
    // r.HandleFunc("/posts", createPost).Methods("POST")
    // r.HandleFunc("/comments/{post_id}", getComments).Methods("GET")
    // r.HandleFunc("/comments/{post_id}", createComment).Methods("POST")

    fmt.Println("Server starting on :8080")
    fmt.Println("Try: http://localhost:8080/posts?limit=20")
    http.ListenAndServe(":8080", r)
}
```

---

## Your Tasks

### Task 1: Implement Cursor Encoding/Decoding

Implement:
- `encodeCursor(cursor interface{}) string`
- `decodeIDCursor(cursorStr string) (IDCursor, error)`
- `decodeTimeCursor(cursorStr string) (TimeCursor, error)`

Use base64 encoding for cursor strings.

### Task 2: ID-Based Cursor Pagination

Implement `getPosts`:
- Parse `cursor` and `limit` parameters
- Decode cursor to get `afterID`
- Query posts WHERE `id > afterID`
- Fetch `limit + 1` items (to check has_more)
- Generate next cursor from last item
- Return paginated response

### Task 3: Time-Based Cursor Pagination

Implement `getPostsFeed`:
- Parse time-based cursor
- Query posts WHERE `created_at < cursor_time`
- Sort by `created_at DESC`
- Use `created_at + id` as composite cursor (for tiebreaking)

### Task 4: Create Post

Implement `createPost`:
- Add new post with current timestamp
- Demonstrate that new posts don't appear in existing pagination
- Return created post

### Task 5: Comment Pagination

Implement `getComments`:
- Cursor pagination for comments on a post
- Order by `created_at` (oldest first)

### Task 6: Bidirectional Pagination

Add support for `previous` cursor:
- Accept `before` parameter
- Query posts WHERE `id < beforeID`
- Order in reverse
- Include `previous` cursor in response

---

## Testing Your Implementation

### Test Basic Cursor Pagination

```bash
# First page (no cursor)
curl "http://localhost:8080/posts?limit=20"

# Response includes next_cursor
# Use it for next page
curl "http://localhost:8080/posts?cursor=eyJpZCI6MjB9&limit=20"
```

### Test Consistency During Writes

```bash
# Terminal 1: Get first page
curl "http://localhost:8080/posts?limit=20" > page1.json

# Terminal 2: Create new post
curl -X POST "http://localhost:8080/posts" \
  -H "Content-Type: application/json" \
  -d '{"title":"New Post","content":"Content","author":"alice"}'

# Terminal 1: Get second page (should be consistent)
CURSOR=$(jq -r '.cursors.next' page1.json)
curl "http://localhost:8080/posts?cursor=$CURSOR&limit=20"
```

**Expected**: New post doesn't appear in pagination sequence.

### Test Time-Based Feed

```bash
# Get recent posts
curl "http://localhost:8080/posts/feed?limit=10"

# Get older posts
curl "http://localhost:8080/posts/feed?cursor=<cursor>&limit=10"
```

### Test Infinite Scroll Simulation

```bash
# Simulate loading multiple pages
#!/bin/bash
CURSOR=""
for i in {1..10}; do
  if [ -z "$CURSOR" ]; then
    RESPONSE=$(curl -s "http://localhost:8080/posts?limit=20")
  else
    RESPONSE=$(curl -s "http://localhost:8080/posts?cursor=$CURSOR&limit=20")
  fi
  
  echo "Page $i"
  echo $RESPONSE | jq '.data | length'
  
  HAS_MORE=$(echo $RESPONSE | jq -r '.has_more')
  if [ "$HAS_MORE" = "false" ]; then
    echo "Reached end"
    break
  fi
  
  CURSOR=$(echo $RESPONSE | jq -r '.cursors.next')
done
```

### Test Comment Pagination

```bash
# Get comments for post
curl "http://localhost:8080/comments/10?limit=5"

# Get next page of comments
curl "http://localhost:8080/comments/10?cursor=<cursor>&limit=5"
```

---

## Expected Behaviors

### Consistency Test

```bash
# Scenario: User loads page 1, new item created, user loads page 2
# Page 1: Items 1-20
# NEW ITEM CREATED (ID 201)
# Page 2: Items 21-40 (NOT 2-21)
```

**Cursor pagination ensures**: Item 21 doesn't move to page 1.

### Performance Test

```bash
# Deep pagination with offset (slow)
GET /products?page=10000&limit=20
# Query: OFFSET 200000 LIMIT 20 (scans 200k rows)

# Deep pagination with cursor (fast)
GET /products?cursor=eyJpZCI6MjAwMDAwfQ&limit=20
# Query: WHERE id > 200000 LIMIT 20 (uses index)
```

---

## Bonus Challenges

### Bonus 1: Bidirectional Pagination

Support both forward and backward:
```go
GET /posts?before=<cursor>&limit=20  // Previous page
GET /posts?after=<cursor>&limit=20   // Next page
```

### Bonus 2: Composite Cursor

Use multiple fields for sorting:
```go
type CompositeCursor struct {
    ViewCount int       `json:"view_count"`
    CreatedAt time.Time `json:"created_at"`
    ID        int       `json:"id"`
}

// Sort by view_count DESC, created_at DESC, id ASC
```

### Bonus 3: Cursor Expiration

Add expiration to cursors:
```go
type ExpiringCursor struct {
    ID        int       `json:"id"`
    ExpiresAt time.Time `json:"expires_at"`
}

// Validate cursor hasn't expired
if time.Now().After(cursor.ExpiresAt) {
    return error("Cursor expired")
}
```

### Bonus 4: Cursor Encryption

Encrypt cursors to prevent manipulation:
```go
// Instead of base64, use AES encryption
encryptedCursor := encrypt(cursorData, secretKey)
```

### Bonus 5: GraphQL Relay Style

Implement Relay cursor connections:
```json
{
  "edges": [
    {
      "node": {...},
      "cursor": "..."
    }
  ],
  "pageInfo": {
    "hasNextPage": true,
    "hasPreviousPage": false,
    "startCursor": "...",
    "endCursor": "..."
  }
}
```

---

## Hints

### Hint 1: Encoding Cursors

```go
func encodeCursor(cursor interface{}) string {
    data, _ := json.Marshal(cursor)
    return base64.StdEncoding.EncodeToString(data)
}

func decodeIDCursor(cursorStr string) (IDCursor, error) {
    data, err := base64.StdEncoding.DecodeString(cursorStr)
    if err != nil {
        return IDCursor{}, err
    }
    
    var cursor IDCursor
    err = json.Unmarshal(data, &cursor)
    return cursor, err
}
```

### Hint 2: Query After Cursor

```go
func queryPostsAfter(afterID, limit int) []Post {
    storageMu.RLock()
    defer storageMu.RUnlock()
    
    result := []Post{}
    
    // Create sorted list
    ids := []int{}
    for id := range posts {
        if id > afterID {
            ids = append(ids, id)
        }
    }
    sort.Ints(ids)
    
    // Take first `limit` items
    for _, id := range ids {
        if len(result) >= limit {
            break
        }
        result = append(result, posts[id])
    }
    
    return result
}
```

### Hint 3: Has More Check

```go
// Fetch limit + 1
items := queryPostsAfter(afterID, limit+1)

// Check if there are more
hasMore := len(items) > limit
if hasMore {
    items = items[:limit]  // Trim extra item
}

// Generate next cursor
var nextCursor string
if hasMore && len(items) > 0 {
    lastItem := items[len(items)-1]
    nextCursor = encodeCursor(IDCursor{ID: lastItem.ID})
}
```

---

## What You're Learning

✅ **Cursor-based pagination** for large datasets  
✅ **Base64 encoding** for cursor strings  
✅ **Consistent pagination** during concurrent writes  
✅ **Infinite scroll** support  
✅ **Time-based cursors** for feeds  
✅ **Composite cursors** for complex sorting  
✅ **Bidirectional pagination** (next/previous)  
✅ **Performance optimization** (WHERE vs OFFSET)  

This is essential for high-scale, real-time applications!
