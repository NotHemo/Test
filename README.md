# Forum Project (Go + WebSockets)

## 🧠 Overview

This project is a real-time forum system with: - Authentication
(Register/Login) - Posts & Comments - Private Messaging (Real-time using
WebSockets)

Tech Stack: - Go (Backend) - SQLite (Database) - JavaScript (Frontend) -
HTML/CSS (UI)

------------------------------------------------------------------------

## 📁 Clean Project Architecture

    forum/
    │
    ├── main.go
    │
    ├── db/
    │   └── sqlite.go
    │
    ├── models/
    │   ├── user.go
    │   ├── post.go
    │   ├── comment.go
    │   └── message.go
    │
    ├── handlers/
    │   ├── auth.go
    │   ├── post.go
    │   └── websocket.go
    │
    ├── services/
    │   ├── auth_service.go
    │   ├── post_service.go
    │   └── chat_service.go
    │
    ├── middleware/
    │   └── session.go
    │
    ├── websocket/
    │   ├── hub.go
    │   └── client.go
    │
    ├── static/
    │   ├── index.html
    │   ├── style.css
    │   └── app.js
    │
    └── utils/
        ├── hash.go
        └── helpers.go

------------------------------------------------------------------------

## 🔥 Responsibilities

-   main.go → start server, routes
-   db → database connection
-   models → data structure
-   handlers → HTTP endpoints
-   services → business logic
-   middleware → authentication checks
-   websocket → real-time system
-   static → frontend

------------------------------------------------------------------------

## 🛠️ Roadmap

### Phase 1 --- Setup

-   Initialize Go module
-   Setup SQLite
-   Create tables

### Phase 2 --- Authentication

-   Register user
-   Login user
-   Hash passwords (bcrypt)
-   Session cookies

### Phase 3 --- Posts & Comments

-   Create post
-   View posts
-   Add comments

### Phase 4 --- Frontend (Basic)

-   Single HTML page
-   JS handles views (login/feed)

### Phase 5 --- WebSockets

-   Setup connection
-   Create Hub & Client
-   Handle connections

### Phase 6 --- Messaging

-   Send/receive messages
-   Store in DB
-   Real-time updates

### Phase 7 --- Improvements

-   Online users list
-   Pagination (messages)
-   UI cleanup

------------------------------------------------------------------------

## ⚠️ Common Mistakes

-   Mixing logic inside handlers
-   Not hashing passwords
-   Starting WebSockets too early
-   No separation of concerns

------------------------------------------------------------------------

## 🚀 First Steps

1.  Run `go mod init forum`
2.  Install dependencies
3.  Setup DB
4.  Build register/login
5.  Test using Postman or browser

------------------------------------------------------------------------

## 🎯 Goal

A clean, scalable real-time forum system using Go.
