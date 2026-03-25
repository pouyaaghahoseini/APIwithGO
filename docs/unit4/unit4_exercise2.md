# Unit 4 - Exercise 2: Social Media API Evolution (v1 → v2 → v3)

**Difficulty**: Advanced  
**Estimated Time**: 60-75 minutes  
**Concepts Covered**: Progressive versioning, adapter pattern, feature flags, version analytics

---

## Objective

Build a social media API that evolves through three versions, showing realistic API evolution:
- **V1**: Basic posts with simple structure
- **V2**: Added reactions, better user info
- **V3**: Added media support, mentions, hashtags

Learn how to manage multiple versions, track usage, and help clients migrate.

---

## API Evolution Story

### V1: Initial Release (Simple Posts)

```go
type PostV1 struct {
    ID        int       `json:"id"`
    Author    string    `json:"author"`  // Just username
    Content   string    `json:"content"`
    Likes     int       `json:"likes"`   // Simple counter
    CreatedAt time.Time `json:"created_at"`
}
```

**Limitations**:
- No author details
- Only "likes", no other reactions
- No media support

---

### V2: Enhanced Engagement (6 months later)

**Breaking Changes**:
- Author is now an object (not string)
- Likes → Reactions (multiple types)

```go
type PostV2 struct {
    ID        int       `json:"id"`
    Author    Author    `json:"author"`  // Full author object
    Content   string    `json:"content"`
    Reactions Reactions `json:"reactions"`  // Multiple reaction types
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
```

---

### V3: Rich Content (1 year later)

**Breaking Changes**:
- Added media attachments
- Added mentions
- Added hashtags
- Reactions → ReactionsV3 (with user lists)

```go
type PostV3 struct {
    ID          int          `json:"id"`
    Author      Author       `json:"author"`
    Content     string       `json:"content"`
    Media       []Media      `json:"media"`
    Mentions    []string     `json:"mentions"`
    Hashtags    []string     `json:"hashtags"`
    Reactions   ReactionsV3  `json:"reactions"`
    CommentCount int         `json:"comment_count"`
    ShareCount  int          `json:"share_count"`
    CreatedAt   time.Time    `json:"created_at"`
}

type Media struct {
    Type string `json:"type"` // "image", "video"
    URL  string `json:"url"`
}

type ReactionsV3 struct {
    Summary map[string]int      `json:"summary"`
    Details map[string][]string `json:"details"` // reaction -> usernames
}
```

---

## Requirements

### Endpoints to Implement

| Version | Method | Path | Description |
|---------|--------|------|-------------|
| V1 | GET | /api/v1/posts | List posts (V1 format) |
| V1 | POST | /api/v1/posts | Create post (V1) |
| V1 | POST | /api/v1/posts/{id}/like | Like a post |
| V2 | GET | /api/v2/posts | List posts (V2 format) |
| V2 | POST | /api/v2/posts | Create post (V2) |
| V2 | POST | /api/v2/posts/{id}/react | Add reaction |
| V3 | GET | /api/v3/posts | List posts (V3 format) |
| V3 | POST | /api/v3/posts | Create post with media |
| V3 | POST | /api/v3/posts/{id}/react | React with details |

### Shared Database Model

```go
type PostRecord struct {
    ID           int
    AuthorID     int
    Content      string
    MediaURLs    []string
    Mentions     []string
    Hashtags     []string
    Reactions    map[string][]string  // reaction type -> list of usernames
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
```

### Special Requirements

1. **Version Analytics**: Track which version each request uses
2. **Migration Hints**: V1 and V2 should suggest upgrading
3. **Adapter Pattern**: V1 and V2 handlers can call V3 logic internally
4. **Feature Flags**: Some V3 features can be enabled in V2
5. **Deprecation Timeline**: V1 deprecated, V2 current, V3 latest

---

## Starter Code

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
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

// Storage
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

// TODO: Implement V1 handlers
func getPostsV1(w http.ResponseWriter, r *http.Request) {}
func createPostV1(w http.ResponseWriter, r *http.Request) {}
func likePostV1(w http.ResponseWriter, r *http.Request) {}

// TODO: Implement V2 handlers
func getPostsV2(w http.ResponseWriter, r *http.Request) {}
func createPostV2(w http.ResponseWriter, r *http.Request) {}
func reactToPostV2(w http.ResponseWriter, r *http.Request) {}

// TODO: Implement V3 handlers
func getPostsV3(w http.ResponseWriter, r *http.Request) {}
func createPostV3(w http.ResponseWriter, r *http.Request) {}
func reactToPostV3(w http.ResponseWriter, r *http.Request) {}

// TODO: Implement conversion functions
func postRecordToV1(post PostRecord, author UserRecord) PostV1 {}
func postRecordToV2(post PostRecord, author UserRecord) PostV2 {}
func postRecordToV3(post PostRecord, author UserRecord) PostV3 {}

// TODO: Implement analytics middleware
func analyticsMiddleware(version string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Track version usage
            next.ServeHTTP(w, r)
        })
    }
}

// TODO: Implement migration hint middleware
func migrationHintMiddleware(currentVersion, nextVersion string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Add headers suggesting upgrade
            next.ServeHTTP(w, r)
        })
    }
}

// Admin endpoint to view analytics
func getVersionStats(w http.ResponseWriter, r *http.Request) {
    statsMu.Lock()
    defer statsMu.Unlock()
    
    respondJSON(w, http.StatusOK, versionStats)
}

// Helpers
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}

// TODO: Extract mentions and hashtags from content
func extractMentions(content string) []string {
    // Find all @username
}

func extractHashtags(content string) []string {
    // Find all #hashtag
}

func main() {
    // TODO: Seed database with users and posts
    
    r := mux.NewRouter()
    
    // TODO: Register V1, V2, V3 routes
    // TODO: Apply appropriate middleware to each version
    
    // Analytics endpoint
    r.HandleFunc("/admin/stats", getVersionStats).Methods("GET")
    
    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", r)
}
```

---

## Sample Data

```go
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
```

---

## Testing Scenarios

### Test All Versions

```bash
# V1 - Get posts (simple format)
curl http://localhost:8080/api/v1/posts

# V2 - Get posts (with reactions)
curl http://localhost:8080/api/v2/posts

# V3 - Get posts (with media, mentions, hashtags)
curl http://localhost:8080/api/v3/posts
```

### Test Progressive Features

```bash
# V1 - Like a post
curl -X POST http://localhost:8080/api/v1/posts/1/like \
  -H "Content-Type: application/json" \
  -d '{"username":"bob"}'

# V2 - React to a post (multiple types)
curl -X POST http://localhost:8080/api/v2/posts/1/react \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","reaction":"love"}'

# V3 - React with full details
curl -X POST http://localhost:8080/api/v3/posts/1/react \
  -H "Content-Type: application/json" \
  -d '{"username":"charlie","reaction":"wow"}'
```

### Test Content Features

```bash
# V3 - Create post with mentions and hashtags
curl -X POST http://localhost:8080/api/v3/posts \
  -H "Content-Type: application/json" \
  -d '{
    "author_id": 1,
    "content": "Great discussion with @bob about #golang!",
    "media": [
      {"type":"image","url":"https://example.com/image.jpg"}
    ]
  }'
```

### Check Analytics

```bash
curl http://localhost:8080/admin/stats

# Expected:
# {
#   "v1": 5,
#   "v2": 12,
#   "v3": 28
# }
```

---

## Requirements

### Version Headers

**V1** (Deprecated):
```
Deprecation: true
Sunset: 2024-06-30
Link: </api/v2/posts>; rel="successor-version"
X-Migration-Guide: https://docs.api.com/v1-to-v2
```

**V2** (Current):
```
X-Upgrade-Available: v3
X-New-Features: media,mentions,hashtags,enhanced-reactions
```

**V3** (Latest):
```
X-API-Version: 3.0
```

### Conversion Rules

**PostRecord → V1**:
- Author: use username only
- Likes: count all reactions

**PostRecord → V2**:
- Author: full author object
- Reactions: count by type

**PostRecord → V3**:
- Include everything
- Auto-extract mentions/hashtags if not stored

---

## Bonus Challenges

### Bonus 1: Version Negotiation
Allow clients to request format:
```bash
GET /api/posts
Accept: application/vnd.socialapi.v2+json
```

### Bonus 2: Gradual Rollout
Add feature flags:
- `?beta=media` - Enable V3 media in V2
- `?beta=reactions` - Enable V2 reactions in V1

### Bonus 3: Usage Metrics
Track more detailed analytics:
- Requests per endpoint per version
- Average response time by version
- Error rate by version

### Bonus 4: Automatic Migration
Add migration endpoint:
```bash
POST /api/migrate/v1-to-v2
# Migrates all V1 client tokens to V2
```

### Bonus 5: Compatibility Layer
Make V1 endpoints use V3 internally (adapter pattern)

---

## Hints

### Hint 1: Extract Mentions/Hashtags

```go
import "regexp"

func extractMentions(content string) []string {
    re := regexp.MustCompile(`@(\w+)`)
    matches := re.FindAllStringSubmatch(content, -1)
    
    mentions := []string{}
    for _, match := range matches {
        if len(match) > 1 {
            mentions = append(mentions, match[1])
        }
    }
    return mentions
}

func extractHashtags(content string) []string {
    re := regexp.MustCompile(`#(\w+)`)
    matches := re.FindAllStringSubmatch(content, -1)
    
    hashtags := []string{}
    for _, match := range matches {
        if len(match) > 1 {
            hashtags = append(hashtags, match[1])
        }
    }
    return hashtags
}
```

### Hint 2: Analytics Tracking

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

### Hint 3: Conversion to V2

```go
func postRecordToV2(post PostRecord, author UserRecord) PostV2 {
    reactions := Reactions{}
    total := 0
    
    for reactionType, users := range post.Reactions {
        count := len(users)
        total += count
        
        switch reactionType {
        case "like":
            reactions.Like = count
        case "love":
            reactions.Love = count
        case "haha":
            reactions.Haha = count
        case "wow":
            reactions.Wow = count
        }
    }
    
    reactions.Total = total
    
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
```

---

## What You're Learning

✅ Managing three concurrent API versions  
✅ Progressive API evolution over time  
✅ Adapter pattern (old versions using new logic)  
✅ Version usage analytics  
✅ Migration hints and guides  
✅ Feature extraction (mentions, hashtags)  
✅ Complex model conversions  
✅ Real-world API lifecycle management  

This exercise shows how APIs evolve in production over years!
