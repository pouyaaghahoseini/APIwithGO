# Unit 4 - Exercise 2 Solution: Social Media API Evolution (v1 → v2 → v3)

**Complete implementation with explanations**

---

## Full Solution Code

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "regexp"
    "strconv"
    "strings"
    "sync"
    "time"

    "github.com/gorilla/mux"
)

// =============================================================================
// V1 MODELS
// =============================================================================

type PostV1 struct {
    ID        int       `json:"id"`
    Author    string    `json:"author"`
    Content   string    `json:"content"`
    Likes     int       `json:"likes"`
    CreatedAt time.Time `json:"created_at"`
}

// =============================================================================
// V2 MODELS
// =============================================================================

type PostV2 struct {
    ID        int       `json:"id"`
    Author    Author    `json:"author"`
    Content   string    `json:"content"`
    Reactions Reactions `json:"reactions"`
    CreatedAt time.Time `json:"created_at"`
}

type Author struct {
    Username    string `json:"username"`
    DisplayName string `json:"display_name"`
    AvatarURL   string `json:"avatar_url"`
}

type Reactions struct {
    Like  int `json:"like"`
    Love  int `json:"love"`
    Haha  int `json:"haha"`
    Wow   int `json:"wow"`
    Total int `json:"total"`
}

// =============================================================================
// V3 MODELS
// =============================================================================

type PostV3 struct {
    ID           int          `json:"id"`
    Author       Author       `json:"author"`
    Content      string       `json:"content"`
    Media        []Media      `json:"media"`
    Mentions     []string     `json:"mentions"`
    Hashtags     []string     `json:"hashtags"`
    Reactions    ReactionsV3  `json:"reactions"`
    CommentCount int          `json:"comment_count"`
    ShareCount   int          `json:"share_count"`
    CreatedAt    time.Time    `json:"created_at"`
}

type Media struct {
    Type string `json:"type"`
    URL  string `json:"url"`
}

type ReactionsV3 struct {
    Summary map[string]int      `json:"summary"`
    Details map[string][]string `json:"details"`
}

// =============================================================================
// DATABASE MODELS
// =============================================================================

type PostRecord struct {
    ID           int
    AuthorID     int
    Content      string
    MediaURLs    []string
    Mentions     []string
    Hashtags     []string
    Reactions    map[string][]string
    CommentCount int
    ShareCount   int
    CreatedAt    time.Time
}

type UserRecord struct {
    ID          int
    Username    string
    DisplayName string
    AvatarURL   string
}

// =============================================================================
// STORAGE
// =============================================================================

var (
    posts      = make(map[int]PostRecord)
    users      = make(map[int]UserRecord)
    nextPostID = 1
    postsMu    sync.RWMutex
    usersMu    sync.RWMutex

    // Analytics
    versionStats = make(map[string]int)
    statsMu      sync.Mutex
)

// =============================================================================
// V1 HANDLERS
// =============================================================================

func getPostsV1(w http.ResponseWriter, r *http.Request) {
    postsMu.RLock()
    usersMu.RLock()
    defer postsMu.RUnlock()
    defer usersMu.RUnlock()

    v1Posts := make([]PostV1, 0, len(posts))
    for _, post := range posts {
        author := users[post.AuthorID]
        v1Posts = append(v1Posts, postRecordToV1(post, author))
    }

    respondJSON(w, http.StatusOK, v1Posts)
}

func createPostV1(w http.ResponseWriter, r *http.Request) {
    var req struct {
        AuthorID int    `json:"author_id"`
        Content  string `json:"content"`
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

    usersMu.RLock()
    author, exists := users[req.AuthorID]
    usersMu.RUnlock()

    if !exists {
        respondError(w, http.StatusBadRequest, "Invalid author_id")
        return
    }

    // Create post
    now := time.Now()
    postsMu.Lock()
    post := PostRecord{
        ID:        nextPostID,
        AuthorID:  req.AuthorID,
        Content:   req.Content,
        Reactions: make(map[string][]string),
        CreatedAt: now,
    }
    posts[nextPostID] = post
    nextPostID++
    postsMu.Unlock()

    // Return V1 format
    v1Post := postRecordToV1(post, author)
    respondJSON(w, http.StatusCreated, v1Post)
}

func likePostV1(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    var req struct {
        Username string `json:"username"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    postsMu.Lock()
    post, exists := posts[id]
    if !exists {
        postsMu.Unlock()
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    // Add like reaction
    if post.Reactions == nil {
        post.Reactions = make(map[string][]string)
    }
    post.Reactions["like"] = append(post.Reactions["like"], req.Username)
    posts[id] = post
    postsMu.Unlock()

    usersMu.RLock()
    author := users[post.AuthorID]
    usersMu.RUnlock()

    v1Post := postRecordToV1(post, author)
    respondJSON(w, http.StatusOK, v1Post)
}

// =============================================================================
// V2 HANDLERS
// =============================================================================

func getPostsV2(w http.ResponseWriter, r *http.Request) {
    postsMu.RLock()
    usersMu.RLock()
    defer postsMu.RUnlock()
    defer usersMu.RUnlock()

    v2Posts := make([]PostV2, 0, len(posts))
    for _, post := range posts {
        author := users[post.AuthorID]
        v2Posts = append(v2Posts, postRecordToV2(post, author))
    }

    respondJSON(w, http.StatusOK, v2Posts)
}

func createPostV2(w http.ResponseWriter, r *http.Request) {
    var req struct {
        AuthorID int    `json:"author_id"`
        Content  string `json:"content"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    if req.Content == "" {
        respondError(w, http.StatusBadRequest, "Content is required")
        return
    }

    usersMu.RLock()
    author, exists := users[req.AuthorID]
    usersMu.RUnlock()

    if !exists {
        respondError(w, http.StatusBadRequest, "Invalid author_id")
        return
    }

    now := time.Now()
    postsMu.Lock()
    post := PostRecord{
        ID:        nextPostID,
        AuthorID:  req.AuthorID,
        Content:   req.Content,
        Reactions: make(map[string][]string),
        CreatedAt: now,
    }
    posts[nextPostID] = post
    nextPostID++
    postsMu.Unlock()

    v2Post := postRecordToV2(post, author)
    respondJSON(w, http.StatusCreated, v2Post)
}

func reactToPostV2(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    var req struct {
        Username string `json:"username"`
        Reaction string `json:"reaction"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    // Validate reaction type
    validReactions := map[string]bool{
        "like": true, "love": true, "haha": true, "wow": true,
    }
    if !validReactions[req.Reaction] {
        respondError(w, http.StatusBadRequest, "Invalid reaction type")
        return
    }

    postsMu.Lock()
    post, exists := posts[id]
    if !exists {
        postsMu.Unlock()
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    if post.Reactions == nil {
        post.Reactions = make(map[string][]string)
    }
    post.Reactions[req.Reaction] = append(post.Reactions[req.Reaction], req.Username)
    posts[id] = post
    postsMu.Unlock()

    usersMu.RLock()
    author := users[post.AuthorID]
    usersMu.RUnlock()

    v2Post := postRecordToV2(post, author)
    respondJSON(w, http.StatusOK, v2Post)
}

// =============================================================================
// V3 HANDLERS
// =============================================================================

func getPostsV3(w http.ResponseWriter, r *http.Request) {
    postsMu.RLock()
    usersMu.RLock()
    defer postsMu.RUnlock()
    defer usersMu.RUnlock()

    v3Posts := make([]PostV3, 0, len(posts))
    for _, post := range posts {
        author := users[post.AuthorID]
        v3Posts = append(v3Posts, postRecordToV3(post, author))
    }

    respondJSON(w, http.StatusOK, v3Posts)
}

func createPostV3(w http.ResponseWriter, r *http.Request) {
    var req struct {
        AuthorID int     `json:"author_id"`
        Content  string  `json:"content"`
        Media    []Media `json:"media"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    if req.Content == "" {
        respondError(w, http.StatusBadRequest, "Content is required")
        return
    }

    usersMu.RLock()
    author, exists := users[req.AuthorID]
    usersMu.RUnlock()

    if !exists {
        respondError(w, http.StatusBadRequest, "Invalid author_id")
        return
    }

    // Extract mentions and hashtags
    mentions := extractMentions(req.Content)
    hashtags := extractHashtags(req.Content)

    // Extract media URLs
    mediaURLs := make([]string, len(req.Media))
    for i, m := range req.Media {
        mediaURLs[i] = m.URL
    }

    now := time.Now()
    postsMu.Lock()
    post := PostRecord{
        ID:        nextPostID,
        AuthorID:  req.AuthorID,
        Content:   req.Content,
        MediaURLs: mediaURLs,
        Mentions:  mentions,
        Hashtags:  hashtags,
        Reactions: make(map[string][]string),
        CreatedAt: now,
    }
    posts[nextPostID] = post
    nextPostID++
    postsMu.Unlock()

    v3Post := postRecordToV3(post, author)
    respondJSON(w, http.StatusCreated, v3Post)
}

func reactToPostV3(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    var req struct {
        Username string `json:"username"`
        Reaction string `json:"reaction"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()

    postsMu.Lock()
    post, exists := posts[id]
    if !exists {
        postsMu.Unlock()
        respondError(w, http.StatusNotFound, "Post not found")
        return
    }

    if post.Reactions == nil {
        post.Reactions = make(map[string][]string)
    }
    post.Reactions[req.Reaction] = append(post.Reactions[req.Reaction], req.Username)
    posts[id] = post
    postsMu.Unlock()

    usersMu.RLock()
    author := users[post.AuthorID]
    usersMu.RUnlock()

    v3Post := postRecordToV3(post, author)
    respondJSON(w, http.StatusOK, v3Post)
}

// =============================================================================
// CONVERSION FUNCTIONS
// =============================================================================

func postRecordToV1(post PostRecord, author UserRecord) PostV1 {
    // Count total reactions as "likes"
    totalLikes := 0
    for _, users := range post.Reactions {
        totalLikes += len(users)
    }

    return PostV1{
        ID:        post.ID,
        Author:    author.Username,
        Content:   post.Content,
        Likes:     totalLikes,
        CreatedAt: post.CreatedAt,
    }
}

func postRecordToV2(post PostRecord, author UserRecord) PostV2 {
    reactions := Reactions{
        Like: len(post.Reactions["like"]),
        Love: len(post.Reactions["love"]),
        Haha: len(post.Reactions["haha"]),
        Wow:  len(post.Reactions["wow"]),
    }

    reactions.Total = reactions.Like + reactions.Love + reactions.Haha + reactions.Wow

    return PostV2{
        ID: post.ID,
        Author: Author{
            Username:    author.Username,
            DisplayName: author.DisplayName,
            AvatarURL:   author.AvatarURL,
        },
        Content:   post.Content,
        Reactions: reactions,
        CreatedAt: post.CreatedAt,
    }
}

func postRecordToV3(post PostRecord, author UserRecord) PostV3 {
    // Build media array
    media := make([]Media, len(post.MediaURLs))
    for i, url := range post.MediaURLs {
        // Determine type from URL (simplified)
        mediaType := "image"
        if strings.Contains(url, "video") {
            mediaType = "video"
        }
        media[i] = Media{Type: mediaType, URL: url}
    }

    // Build reactions V3
    reactionsV3 := ReactionsV3{
        Summary: make(map[string]int),
        Details: make(map[string][]string),
    }

    for reactionType, users := range post.Reactions {
        reactionsV3.Summary[reactionType] = len(users)
        reactionsV3.Details[reactionType] = users
    }

    return PostV3{
        ID: post.ID,
        Author: Author{
            Username:    author.Username,
            DisplayName: author.DisplayName,
            AvatarURL:   author.AvatarURL,
        },
        Content:      post.Content,
        Media:        media,
        Mentions:     post.Mentions,
        Hashtags:     post.Hashtags,
        Reactions:    reactionsV3,
        CommentCount: post.CommentCount,
        ShareCount:   post.ShareCount,
        CreatedAt:    post.CreatedAt,
    }
}

// =============================================================================
// UTILITY FUNCTIONS
// =============================================================================

func extractMentions(content string) []string {
    re := regexp.MustCompile(`@(\w+)`)
    matches := re.FindAllStringSubmatch(content, -1)

    mentions := []string{}
    seen := make(map[string]bool)
    for _, match := range matches {
        if len(match) > 1 && !seen[match[1]] {
            mentions = append(mentions, match[1])
            seen[match[1]] = true
        }
    }
    return mentions
}

func extractHashtags(content string) []string {
    re := regexp.MustCompile(`#(\w+)`)
    matches := re.FindAllStringSubmatch(content, -1)

    hashtags := []string{}
    seen := make(map[string]bool)
    for _, match := range matches {
        if len(match) > 1 && !seen[match[1]] {
            hashtags = append(hashtags, match[1])
            seen[match[1]] = true
        }
    }
    return hashtags
}

// =============================================================================
// MIDDLEWARE
// =============================================================================

func analyticsMiddleware(version string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            statsMu.Lock()
            versionStats[version]++
            statsMu.Unlock()

            next.ServeHTTP(w, r)
        })
    }
}

func deprecationMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Deprecation", "true")
        w.Header().Set("Sunset", "2024-06-30")
        v2Path := strings.Replace(r.URL.Path, "/v1/", "/v2/", 1)
        w.Header().Set("Link", fmt.Sprintf("<%s>; rel=\"successor-version\"", v2Path))
        w.Header().Set("X-Migration-Guide", "https://docs.api.com/v1-to-v2")

        next.ServeHTTP(w, r)
    })
}

func upgradeHintMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("X-Upgrade-Available", "v3")
        w.Header().Set("X-New-Features", "media,mentions,hashtags,enhanced-reactions")
        w.Header().Set("X-Migration-Guide", "https://docs.api.com/v2-to-v3")

        next.ServeHTTP(w, r)
    })
}

// =============================================================================
// ADMIN HANDLERS
// =============================================================================

func getVersionStats(w http.ResponseWriter, r *http.Request) {
    statsMu.Lock()
    defer statsMu.Unlock()

    respondJSON(w, http.StatusOK, versionStats)
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
    // Seed users
    usersMu.Lock()
    users[1] = UserRecord{
        ID:          1,
        Username:    "alice",
        DisplayName: "Alice Johnson",
        AvatarURL:   "https://example.com/avatars/alice.jpg",
    }
    users[2] = UserRecord{
        ID:          2,
        Username:    "bob",
        DisplayName: "Bob Smith",
        AvatarURL:   "https://example.com/avatars/bob.jpg",
    }
    usersMu.Unlock()

    // Seed posts
    postsMu.Lock()
    posts[1] = PostRecord{
        ID:       1,
        AuthorID: 1,
        Content:  "Hello world! This is my first post.",
        Reactions: map[string][]string{
            "like": {"bob"},
        },
        CreatedAt: time.Now().Add(-24 * time.Hour),
    }

    posts[2] = PostRecord{
        ID:       2,
        AuthorID: 2,
        Content:  "Check out this cool feature! @alice #golang",
        Mentions: []string{"alice"},
        Hashtags: []string{"golang"},
        Reactions: map[string][]string{
            "like": {"alice"},
            "love": {"alice"},
        },
        CreatedAt: time.Now().Add(-2 * time.Hour),
    }

    nextPostID = 3
    postsMu.Unlock()
}

// =============================================================================
// MAIN
// =============================================================================

func main() {
    seedDatabase()

    r := mux.NewRouter()

    // V1 routes (deprecated)
    v1 := r.PathPrefix("/api/v1").Subrouter()
    v1.Use(analyticsMiddleware("v1"))
    v1.Use(deprecationMiddleware)
    v1.HandleFunc("/posts", getPostsV1).Methods("GET")
    v1.HandleFunc("/posts", createPostV1).Methods("POST")
    v1.HandleFunc("/posts/{id}/like", likePostV1).Methods("POST")

    // V2 routes (current)
    v2 := r.PathPrefix("/api/v2").Subrouter()
    v2.Use(analyticsMiddleware("v2"))
    v2.Use(upgradeHintMiddleware)
    v2.HandleFunc("/posts", getPostsV2).Methods("GET")
    v2.HandleFunc("/posts", createPostV2).Methods("POST")
    v2.HandleFunc("/posts/{id}/react", reactToPostV2).Methods("POST")

    // V3 routes (latest)
    v3 := r.PathPrefix("/api/v3").Subrouter()
    v3.Use(analyticsMiddleware("v3"))
    v3.HandleFunc("/posts", getPostsV3).Methods("GET")
    v3.HandleFunc("/posts", createPostV3).Methods("POST")
    v3.HandleFunc("/posts/{id}/react", reactToPostV3).Methods("POST")

    // Admin
    r.HandleFunc("/admin/stats", getVersionStats).Methods("GET")

    fmt.Println("Server starting on :8080")
    fmt.Println("V1 (deprecated): http://localhost:8080/api/v1/posts")
    fmt.Println("V2 (current):    http://localhost:8080/api/v2/posts")
    fmt.Println("V3 (latest):     http://localhost:8080/api/v3/posts")
    fmt.Println("Analytics:       http://localhost:8080/admin/stats")
    http.ListenAndServe(":8080", r)
}
```

---

## Key Concepts Explained

### 1. Progressive Evolution Across 3 Versions

**V1 → V2 Changes**:
- Author: `string` → `Author` object
- Likes: `int` → `Reactions` with types

**V2 → V3 Changes**:
- Added: Media, Mentions, Hashtags
- Reactions: Summary counts → Summary + Details

### 2. Analytics Middleware

```go
func analyticsMiddleware(version string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            statsMu.Lock()
            versionStats[version]++
            statsMu.Unlock()

            next.ServeHTTP(w, r)
        })
    }
}
```

**Tracks**:
- How many requests each version receives
- Helps decide when to deprecate
- Shows migration progress

### 3. Content Extraction (V3 Feature)

```go
func extractMentions(content string) []string {
    re := regexp.MustCompile(`@(\w+)`)
    matches := re.FindAllStringSubmatch(content, -1)
    
    mentions := []string{}
    seen := make(map[string]bool)
    for _, match := range matches {
        if len(match) > 1 && !seen[match[1]] {
            mentions = append(mentions, match[1])
            seen[match[1]] = true
        }
    }
    return mentions
}
```

**Auto-extracts**:
- `@username` → mentions array
- `#hashtag` → hashtags array

### 4. Version Headers

**V1 (Deprecated)**:
```
Deprecation: true
Sunset: 2024-06-30
Link: </api/v2/posts>; rel="successor-version"
X-Migration-Guide: https://docs.api.com/v1-to-v2
```

**V2 (Current, with upgrade hint)**:
```
X-Upgrade-Available: v3
X-New-Features: media,mentions,hashtags,enhanced-reactions
X-Migration-Guide: https://docs.api.com/v2-to-v3
```

**V3 (Latest)**:
- No special headers (current version)

### 5. Complex Conversion: Reactions Evolution

```go
// V1: Just count everything as "likes"
totalLikes := 0
for _, users := range post.Reactions {
    totalLikes += len(users)
}

// V2: Break down by type, show counts
reactions := Reactions{
    Like: len(post.Reactions["like"]),
    Love: len(post.Reactions["love"]),
    Haha: len(post.Reactions["haha"]),
    Wow:  len(post.Reactions["wow"]),
}

// V3: Full details with user lists
reactionsV3 := ReactionsV3{
    Summary: map[string]int{
        "like": len(post.Reactions["like"]),
        "love": len(post.Reactions["love"]),
    },
    Details: map[string][]string{
        "like": post.Reactions["like"],  // ["bob", "alice"]
        "love": post.Reactions["love"],  // ["alice"]
    },
}
```

---

## Response Format Comparison

### Same Post Across All Versions

**V1 Response**:
```json
{
  "id": 2,
  "author": "bob",
  "content": "Check out this cool feature! @alice #golang",
  "likes": 2,
  "created_at": "2024-01-15T10:00:00Z"
}
```

**V2 Response**:
```json
{
  "id": 2,
  "author": {
    "username": "bob",
    "display_name": "Bob Smith",
    "avatar_url": "https://example.com/avatars/bob.jpg"
  },
  "content": "Check out this cool feature! @alice #golang",
  "reactions": {
    "like": 1,
    "love": 1,
    "haha": 0,
    "wow": 0,
    "total": 2
  },
  "created_at": "2024-01-15T10:00:00Z"
}
```

**V3 Response**:
```json
{
  "id": 2,
  "author": {
    "username": "bob",
    "display_name": "Bob Smith",
    "avatar_url": "https://example.com/avatars/bob.jpg"
  },
  "content": "Check out this cool feature! @alice #golang",
  "media": [],
  "mentions": ["alice"],
  "hashtags": ["golang"],
  "reactions": {
    "summary": {
      "like": 1,
      "love": 1
    },
    "details": {
      "like": ["alice"],
      "love": ["alice"]
    }
  },
  "comment_count": 0,
  "share_count": 0,
  "created_at": "2024-01-15T10:00:00Z"
}
```

---

## Testing All Versions

```bash
#!/bin/bash

BASE="http://localhost:8080"

echo "=== V1 (Deprecated) ==="
curl -i $BASE/api/v1/posts | head -20
curl $BASE/api/v1/posts | jq .

echo -e "\n=== V2 (Current) ==="
curl -i $BASE/api/v2/posts | head -20
curl $BASE/api/v2/posts | jq .

echo -e "\n=== V3 (Latest) ==="
curl $BASE/api/v3/posts | jq .

echo -e "\n=== Create Post with Media (V3) ==="
curl -X POST $BASE/api/v3/posts \
  -H "Content-Type: application/json" \
  -d '{
    "author_id": 1,
    "content": "Great discussion with @bob about #golang!",
    "media": [
      {"type":"image","url":"https://example.com/photo.jpg"}
    ]
  }' | jq .

echo -e "\n=== React to Post ==="
# V1: Like
curl -X POST $BASE/api/v1/posts/1/like \
  -H "Content-Type: application/json" \
  -d '{"username":"alice"}' | jq .

# V2: Multiple reaction types
curl -X POST $BASE/api/v2/posts/1/react \
  -H "Content-Type: application/json" \
  -d '{"username":"bob","reaction":"love"}' | jq .

# V3: Same as V2 but with details
curl -X POST $BASE/api/v3/posts/1/react \
  -H "Content-Type: application/json" \
  -d '{"username":"charlie","reaction":"wow"}' | jq .

echo -e "\n=== Analytics ==="
curl $BASE/admin/stats | jq .
```

---

## Migration Timeline

| Date | Version | Status | Action |
|------|---------|--------|--------|
| Jan 2024 | V1 | Deprecated | Add deprecation headers |
| Jan 2024 | V2 | Current | Main version |
| Mar 2024 | V3 | Latest | Launch with upgrade hints |
| Apr 2024 | V1 | Warning | Email users to migrate |
| Jul 2024 | V1 | Removed | Shutdown V1 endpoints |
| Dec 2024 | V2 | Deprecate? | Evaluate V2→V3 migration |

---

## Feature Matrix

| Feature | V1 | V2 | V3 |
|---------|----|----|-----|
| Author Info | Username only | Full object | Full object |
| Reactions | Total count | By type | + User details |
| Media | ❌ | ❌ | ✅ |
| Mentions | ❌ | ❌ | ✅ Auto-extract |
| Hashtags | ❌ | ❌ | ✅ Auto-extract |
| Counts | Likes only | Reactions | + Comments, Shares |
| Analytics | ✅ Tracked | ✅ Tracked | ✅ Tracked |

---

## What You've Learned

✅ **Progressive versioning** across 3 versions  
✅ **Analytics tracking** for version usage  
✅ **Feature extraction** (mentions, hashtags with regex)  
✅ **Complex model evolution** (reactions V1→V2→V3)  
✅ **Migration hints** via headers  
✅ **Deprecation timeline** management  
✅ **Shared database** supporting all versions  
✅ **Real-world API lifecycle** over time  

This demonstrates how APIs evolve in production over multiple years!
