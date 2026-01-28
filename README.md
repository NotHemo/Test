# My Contributions to Forum Project

## 🎯 Overview

I was responsible for two critical parts of the forum: the **session management system** that handles user authentication, and the **complete posts system** which is the core feature of the forum.

---

## Part 1: Session Management System

### What I Built

I implemented the complete session management system that handles user authentication state across the forum. This includes creating sessions on login, validating sessions on each request, and managing session lifecycle.

### Key Features

#### 1. Session Creation (`session.go` lines 14-26)

- **UUID v4 Generation**: Uses cryptographically secure random tokens
- **24-Hour Expiration**: Sessions automatically expire for security
- **One Session Per User**: Old sessions are deleted when user logs in again
- **Database Persistence**: Sessions stored in SQLite with user_id, token, and expiration time

```go
func CreateSession(db *sql.DB, UserID int) (string, error) {
	// Delete old sessions - enforce one session per user
	_, err := db.Exec(`DELETE FROM sessions WHERE user_id = ?`, UserID)
	if err != nil {
		return "", err
	}

	sessionID := uuid.New().String()
	expiresAt := time.Now().Add(24 * time.Hour)

	_, err = db.Exec(
		`INSERT INTO sessions (session_token, user_id, expires_at) VALUES (?, ?, ?)`,
		sessionID, UserID, expiresAt,
	)
	if err != nil {
		return "", err
	}
	return sessionID, nil
}
```

#### 2. Cookie Management (`session.go` lines 27-35)

- **HttpOnly Flag**: Prevents JavaScript/XSS attacks from stealing tokens
- **Secure Flag**: (Can be added) Requires HTTPS in production
- **24-Hour Lifespan**: Matches database expiration for consistency
- **Standard Cookie Name**: `session_id` for clarity

```go
func SetSessionCookie(w http.ResponseWriter, sessionID string) {
	cookie := http.Cookie{
		Name:     "session_id",
		Value:    sessionID,
		Path:     "/",
		HttpOnly: true,
		Expires:  time.Now().Add(24 * time.Hour),
	}
	http.SetCookie(w, &cookie)
}
```

#### 3. Session Validation (`session.go` lines 37-60)

- **Used by Every Protected Route**: Checks authorization before allowing actions
- **Cookie Verification**: Retrieves and validates cookie from request
- **Database Lookup**: Confirms session exists and belongs to valid user
- **Expiration Check**: Returns false if session has expired
- **Returns User ID**: Allows handlers to know who the user is

```go
func GetUserFromSession(r *http.Request, db *sql.DB) (int, bool) {
	cookie, err := r.Cookie("session_id")
	if err != nil {
		return 0, false
	}

	var UserID int
	var expiresAt time.Time

	err = db.QueryRow(
		`SELECT user_id, expires_at FROM sessions WHERE session_token = ?`,
		cookie.Value,
	).Scan(&UserID, &expiresAt)

	if err != nil {
		return 0, false
	}

	if time.Now().After(expiresAt) {
		return 0, false
	}
	return UserID, true
}
```

#### 4. Logout Functionality (`session.go` lines 62-79)

- **Database Cleanup**: Deletes session from database
- **Cookie Deletion**: Sets MaxAge: -1 to immediately remove cookie from browser
- **Redirect to Home**: Sends user back to homepage
- **Error Resilience**: Continues even if cookie doesn't exist

```go
func LogoutHandler(w http.ResponseWriter, r *http.Request, db *sql.DB) {
	cookie, err := r.Cookie("session_id")
	if err == nil {
		db.Exec(
			`DELETE FROM sessions WHERE session_token = ?`,
			cookie.Value,
		)
	}

	http.SetCookie(w, &http.Cookie{
		Name:   "session_id",
		Value:  "",
		Path:   "/",
		MaxAge: -1,
	})

	http.Redirect(w, r, "/", http.StatusSeeOther)
}
```

### Integration Points

- Used by `/post/create` - Only logged-in users can create posts
- Used by `/comment/create` - Only logged-in users can comment
- Used by `/reaction` - Only logged-in users can like/dislike
- Used by `HomeHandler` - Shows personalized content based on session

---

## Part 2: Posts System

### What I Built

I built the entire posts feature, which is the core of the forum. This includes post creation with validation, viewing posts with all related data, and integrating categories and reactions.

### Key Features

#### 1. Post Creation Handler (`posts.go` lines 20-165)

**Authentication Check**:
```go
userID, loggedIn := GetUserFromSession(r, db)
if !loggedIn {
    http.Redirect(w, r, "/login", http.StatusSeeOther)
    return
}
```

**GET Request**: Displays form with available categories
```go
case "GET":
    categories := defaultCategories()
    
    data := struct {
        Title              string
        Content            string
        Categories         []string
        SelectedCategories map[string]bool
        Errors             struct {
            Title        string
            Content      string
            GeneralError string
        }
    }{
        Title:              "",
        Content:            "",
        Categories:         categories,
        SelectedCategories: make(map[string]bool),
    }

    err = tmpl.Execute(w, data)
```

**POST Request**: Validates and saves post
- **Title Validation**: Required, max 255 characters
- **Content Validation**: Required, max 5000 characters
- **Category Selection**: At least one category required

```go
errors := struct {
    Title        string
    Content      string
    GeneralError string
}{}

if title == "" {
    errors.Title = "Title is required"
}
if len(title) > 255 {
    errors.Title = "Title is too long (max 255 characters)"
}
if content == "" {
    errors.Content = "Content is required"
}
if len(content) > 5000 {
    errors.Content = "Content is too long (max 5000 characters)"
}
if len(selectedCategoriesSlice) == 0 {
    errors.GeneralError = "Please select at least one category"
}
```

**Database Insert**:
```go
result, err := db.Exec(
    `INSERT INTO posts (user_id, title, content, created_at) 
     VALUES (?, ?, ?, CURRENT_TIMESTAMP)`,
    userID, title, content,
)

postID, err := result.LastInsertId()

// Insert each category
for _, category := range selectedCategoriesSlice {
    _, err := db.Exec(
        `INSERT INTO post_categories (post_id, category) VALUES (?, ?)`,
        postID,
        category,
    )
}
```

#### 2. Category System

**Many-to-Many Relationship**:
- Uses `post_categories` junction table
- Each post can have multiple categories
- Each category can appear on multiple posts
- Flexible and scalable design

**Category List**:
```go
func defaultCategories() []string {
	return []string{
		"General", "Technology", "Sports",
		"Entertainment", "Education",
		"Business", "Travel", "News",
	}
}
```

#### 3. View Post Handler (`posts.go` lines 167-269)

**Fetch Post Data**:
```go
post, err := models.GetPostByID(db, postID)
if err != nil {
    if err == sql.ErrNoRows {
        PostNotFoundHandler(w, r)
        return
    }
}
```

**Fetch Comments**:
```go
comments, err := models.GetCommentsByPost(db, postID)
if err != nil {
    comments = []models.Comment{}
}
```

**Check User Reactions** (if logged in):
```go
var userPostReaction string
if loggedIn {
    userPostReaction, _ = models.GetUserPostReaction(db, userID, postID)
}

commentReactions := make(map[int]string)
if loggedIn {
    for _, comment := range comments {
        reaction, _ := models.GetUserCommentReaction(db, userID, comment.ID)
        commentReactions[comment.ID] = reaction
    }
}
```

**Render Template**:
```go
data := struct {
    IsLoggedIn       bool
    UserID           int
    Username         string
    Post             *models.Post
    Comments         []models.Comment
    UserPostReaction string
    CommentReactions map[int]string
}{
    IsLoggedIn:       loggedIn,
    UserID:           userID,
    Username:         username,
    Post:             post,
    Comments:         comments,
    UserPostReaction: userPostReaction,
    CommentReactions: commentReactions,
}
```

#### 4. Integration Points

- **With Session System**: Uses `GetUserFromSession()` to check authorization
- **With Comments System**: Loads and displays all comments for the post
- **With Reactions System**: Shows like/dislike counts and user's reaction state
- **With Models Layer**: Uses `models.GetPostByID()` for data access (separation of concerns)

---

## 🗣️ How to Present This

### Opening (30 seconds)
"I was responsible for two critical parts of the forum: the session management system that handles user authentication, and the complete posts system which is the core feature of the forum."

### Session System Explanation (2 minutes)
1. Start with the security importance of session management
2. Explain UUID generation and why it's better than sequential IDs
3. Show the one-session-per-user design decision
4. Mention HttpOnly cookies prevent XSS attacks
5. Explain how validation happens on every request
6. "This session system is used by every protected route in the application"

### Posts System Explanation (3 minutes)
1. Explain the creation flow with validation
2. Show how categories work with the junction table pattern
3. Walk through the view post handler bringing everything together
4. Mention how it integrates with comments and reactions
5. "Posts are the foundation that other features build upon"

### Challenges & Solutions (1 minute)
- "Session security: I solved this with HttpOnly cookies and UUID tokens"
- "Category relationships: I used a junction table pattern for flexibility"
- "Edge cases: I handle expired sessions and missing posts with proper error responses"

### Closing (30 seconds)
"Both systems are fully functional and handle edge cases properly. The session system is secure and the posts system is the foundation that other features like comments and reactions build upon."

---

## 💡 Questions to Anticipate

### About Sessions

**Q: "Why UUID instead of just incrementing numbers?"**
A: "UUIDs are cryptographically random and unpredictable. Sequential IDs would let attackers guess valid session tokens. UUID v4 has 122 bits of randomness - practically impossible to guess."

**Q: "Why 24 hours for expiration?"**
A: "It balances security and user convenience. Users stay logged in for a day but sessions don't last indefinitely. We could make this configurable based on requirements."

**Q: "What if someone steals the cookie?"**
A: "HttpOnly prevents JavaScript access, reducing XSS risk. In production, we'd add the Secure flag to require HTTPS, preventing interception. We could also add IP validation or device fingerprinting."

**Q: "Why delete old sessions on login?"**
A: "This prevents session fixation attacks. If someone compromises one session token, the old one is deleted when the user logs in again from another device."

### About Posts

**Q: "Why limit title to 255 characters?"**
A: "That's a database VARCHAR best practice - enough for any reasonable title but prevents abuse. Content has a higher limit of 5000 for the actual post body."

**Q: "How do you handle multiple categories?"**
A: "I use a junction table pattern - `post_categories` has post_id and category columns. This allows many-to-many relationships. When creating a post, I loop through selected categories and insert each relationship."

**Q: "What happens if post creation fails mid-way through categories?"**
A: "Good catch! Currently, if a category insert fails, previous categories are already saved. Ideally, I'd wrap this in a transaction to make it atomic - all or nothing. That's something I'd improve with more time."

**Q: "How do you prevent unauthorized post access?"**
A: "Every handler that needs authentication uses `GetUserFromSession()` which returns false if there's no valid session. If not logged in, we redirect to login page."

**Q: "Why store categories in a separate table?"**
A: "It's more flexible. We can add new categories without modifying the posts table, and we can query 'all posts in Technology' efficiently."

---

## ✅ Key Points to Emphasize

- **Security-First Approach**: UUID tokens, HttpOnly cookies, session expiration
- **Proper Validation**: Input length limits, required fields, edge case handling
- **Database Design**: Junction tables for many-to-many, foreign keys, proper relationships
- **Integration**: Your work connects login, posts, comments, and reactions
- **Error Handling**: 404s for missing posts, validation errors, expired session handling
- **Code Organization**: Handlers, models, clear separation of concerns
- **User Experience**: Sessions persist across page reloads, validation feedback on forms

---

## 🎯 Demo Script

If asked for a demo, follow this flow:

**Step 1: Show Logout/Login Flow**
- Show no session cookie before login
- Explain "can't create posts without authorization"
- Login with credentials
- Show session_id cookie now exists
- Show "now authorized to create posts"

**Step 2: Create a Post**
- Go to "Create Post"
- Try to submit empty form
- Show validation errors appear
- Fill in title, content, select categories
- Submit successfully
- Show redirect to view-post

**Step 3: View the Post**
- Show post displays with author, categories, timestamps
- Show like/dislike counts (0 initially)
- Show "Like/Dislike" buttons available because logged in
- Show comments section with form to add comment

**Step 4: Show Session in Browser DevTools**
- Open DevTools → Application → Cookies
- "Here's the session_id cookie"
- "Notice HttpOnly is checked - JavaScript can't access it"
- Show expiration is 24 hours from login

**Step 5: Test Logout**
- Click Logout
- Show session_id cookie is deleted
- Show redirected to homepage
- Try to create post - redirected to login

---

## 📊 Impact Summary

### What This Enables
- ✅ User authentication and authorization
- ✅ Persistent login state across page reloads
- ✅ Secure session management
- ✅ Core forum feature (creating/viewing posts)
- ✅ Foundation for comments and reactions

### Code Files Involved
- `internal/handlers/session.go` - Session lifecycle management
- `internal/handlers/posts.go` - Post creation and viewing
- `internal/models/posts.go` - Post data access (worked with team)
- `internal/database/forum.sql` - Database schema (sessions & posts tables)
- `internal/models/comments.go` - Comment display (integrated with)
- `internal/models/reactions.go` - Reaction display (integrated with)

### Testing Performed
- ✅ Login/logout flow works correctly
- ✅ Session expires properly
- ✅ Only logged-in users can create posts
- ✅ Post validation prevents invalid data
- ✅ Categories save and display correctly
- ✅ Post view shows all related data (comments, reactions)
- ✅ Unauthorized users redirected to login

---

## 🚀 What I Learned

1. **Session Security**: How to securely manage user state with UUIDs and cookies
2. **Database Design**: Using junction tables for many-to-many relationships
3. **Separation of Concerns**: Keeping handlers separate from models
4. **Input Validation**: Server-side validation is critical
5. **HTTP Best Practices**: Understanding status codes, redirects, and error handling
6. **Go Web Development**: How Go's `net/http` and `database/sql` work together
7. **Cryptographic Security**: Why random tokens matter for session management

---

**Good luck with your presentation! You've built solid, functional systems. 💪**
