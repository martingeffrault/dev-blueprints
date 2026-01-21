# Go Gin (2025)

> **Last updated**: January 2026
> **Versions covered**: Gin 1.10+
> **Purpose**: High-performance HTTP web framework for Go with Martini-like API

---

## Philosophy (2025-2026)

Gin is the **battle-tested Go web framework** — 40× faster than Martini, with an intuitive API. It's the de facto choice for Go REST APIs, balancing simplicity with performance.

**Key philosophical shifts:**
- **Performance first** — httprouter under the hood
- **Crash-free** — Built-in recovery middleware
- **Minimal magic** — Explicit over implicit
- **Middleware-centric** — Composable request handling
- **JSON validation** — Struct tags for validation
- **Route grouping** — Organize related endpoints
- **Zero allocation router** — Memory efficient

---

## TL;DR

- Use `gin.Context` for request/response handling
- Use struct tags for JSON binding and validation
- Group routes with `router.Group()`
- Use middleware for cross-cutting concerns
- Always use `c.ShouldBind*` (not `c.Bind*`) for explicit errors
- Enable `gin.ReleaseMode` in production
- Use `c.Error()` for error collection
- Implement graceful shutdown
- Never expose internal errors to clients

---

## Best Practices

### Project Structure

```
my-gin-app/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── config/
│   │   └── config.go
│   ├── handlers/
│   │   ├── health.go
│   │   └── users.go
│   ├── middleware/
│   │   ├── auth.go
│   │   ├── cors.go
│   │   └── error.go
│   ├── models/
│   │   └── user.go
│   ├── repository/
│   │   └── user.go
│   ├── routes/
│   │   └── routes.go
│   └── services/
│       └── user.go
├── pkg/
│   └── response/
│       └── response.go
├── migrations/
├── go.mod
├── go.sum
└── .env
```

### Main Entry Point

```go
// cmd/server/main.go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/gin-gonic/gin"
    "myapp/internal/config"
    "myapp/internal/routes"
)

func main() {
    // Load configuration
    cfg := config.Load()

    // Set Gin mode
    if cfg.Environment == "production" {
        gin.SetMode(gin.ReleaseMode)
    }

    // Create router
    router := routes.SetupRouter(cfg)

    // Create server
    srv := &http.Server{
        Addr:         ":" + cfg.Port,
        Handler:      router,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // Start server in goroutine
    go func() {
        log.Printf("Server starting on port %s", cfg.Port)
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Failed to start server: %v", err)
        }
    }()

    // Graceful shutdown
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Println("Shutting down server...")

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }

    log.Println("Server exited")
}
```

### Configuration

```go
// internal/config/config.go
package config

import (
    "os"
)

type Config struct {
    Environment string
    Port        string
    DatabaseURL string
    JWTSecret   string
}

func Load() *Config {
    return &Config{
        Environment: getEnv("ENVIRONMENT", "development"),
        Port:        getEnv("PORT", "8080"),
        DatabaseURL: getEnv("DATABASE_URL", "postgres://localhost/myapp"),
        JWTSecret:   getEnv("JWT_SECRET", "secret"),
    }
}

func getEnv(key, fallback string) string {
    if value, ok := os.LookupEnv(key); ok {
        return value
    }
    return fallback
}
```

### Routes Setup

```go
// internal/routes/routes.go
package routes

import (
    "github.com/gin-gonic/gin"
    "myapp/internal/config"
    "myapp/internal/handlers"
    "myapp/internal/middleware"
    "myapp/internal/repository"
    "myapp/internal/services"
)

func SetupRouter(cfg *config.Config) *gin.Engine {
    router := gin.New()

    // Global middleware
    router.Use(gin.Logger())
    router.Use(gin.Recovery())
    router.Use(middleware.CORS())
    router.Use(middleware.ErrorHandler())

    // Initialize dependencies
    db := connectDB(cfg.DatabaseURL)
    userRepo := repository.NewUserRepository(db)
    userService := services.NewUserService(userRepo)
    userHandler := handlers.NewUserHandler(userService)

    // Health check
    router.GET("/health", handlers.HealthCheck)

    // API v1
    v1 := router.Group("/api/v1")
    {
        // Public routes
        v1.POST("/register", userHandler.Register)
        v1.POST("/login", userHandler.Login)

        // Protected routes
        protected := v1.Group("")
        protected.Use(middleware.Auth(cfg.JWTSecret))
        {
            protected.GET("/users", userHandler.List)
            protected.GET("/users/:id", userHandler.Get)
            protected.PUT("/users/:id", userHandler.Update)
            protected.DELETE("/users/:id", userHandler.Delete)
            protected.GET("/me", userHandler.GetCurrentUser)
        }
    }

    return router
}
```

### Standardized Response

```go
// pkg/response/response.go
package response

import (
    "github.com/gin-gonic/gin"
)

type Response struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data,omitempty"`
    Error   *ErrorInfo  `json:"error,omitempty"`
    Meta    *Meta       `json:"meta,omitempty"`
}

type ErrorInfo struct {
    Code    string `json:"code"`
    Message string `json:"message"`
}

type Meta struct {
    Page       int `json:"page,omitempty"`
    PerPage    int `json:"per_page,omitempty"`
    Total      int `json:"total,omitempty"`
    TotalPages int `json:"total_pages,omitempty"`
}

func Success(c *gin.Context, status int, data interface{}) {
    c.JSON(status, Response{
        Success: true,
        Data:    data,
    })
}

func SuccessWithMeta(c *gin.Context, status int, data interface{}, meta *Meta) {
    c.JSON(status, Response{
        Success: true,
        Data:    data,
        Meta:    meta,
    })
}

func Error(c *gin.Context, status int, code, message string) {
    c.JSON(status, Response{
        Success: false,
        Error: &ErrorInfo{
            Code:    code,
            Message: message,
        },
    })
}
```

### Models with Validation

```go
// internal/models/user.go
package models

import (
    "time"
)

type User struct {
    ID        uint      `json:"id" gorm:"primaryKey"`
    Name      string    `json:"name" gorm:"size:100;not null"`
    Email     string    `json:"email" gorm:"uniqueIndex;not null"`
    Password  string    `json:"-" gorm:"not null"` // Never expose
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

// Request DTOs with validation
type CreateUserRequest struct {
    Name     string `json:"name" binding:"required,min=2,max=100"`
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
}

type UpdateUserRequest struct {
    Name  string `json:"name" binding:"omitempty,min=2,max=100"`
    Email string `json:"email" binding:"omitempty,email"`
}

type LoginRequest struct {
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required"`
}

// Response DTOs
type UserResponse struct {
    ID        uint      `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
}

func (u *User) ToResponse() *UserResponse {
    return &UserResponse{
        ID:        u.ID,
        Name:      u.Name,
        Email:     u.Email,
        CreatedAt: u.CreatedAt,
    }
}
```

### Handlers

```go
// internal/handlers/users.go
package handlers

import (
    "net/http"
    "strconv"

    "github.com/gin-gonic/gin"
    "myapp/internal/models"
    "myapp/internal/services"
    "myapp/pkg/response"
)

type UserHandler struct {
    service *services.UserService
}

func NewUserHandler(service *services.UserService) *UserHandler {
    return &UserHandler{service: service}
}

func (h *UserHandler) Register(c *gin.Context) {
    var req models.CreateUserRequest

    // Use ShouldBindJSON for explicit error handling
    if err := c.ShouldBindJSON(&req); err != nil {
        response.Error(c, http.StatusBadRequest, "VALIDATION_ERROR", err.Error())
        return
    }

    user, err := h.service.Create(&req)
    if err != nil {
        if err == services.ErrEmailExists {
            response.Error(c, http.StatusConflict, "EMAIL_EXISTS", "Email already registered")
            return
        }
        response.Error(c, http.StatusInternalServerError, "INTERNAL_ERROR", "Failed to create user")
        return
    }

    response.Success(c, http.StatusCreated, user.ToResponse())
}

func (h *UserHandler) List(c *gin.Context) {
    page, _ := strconv.Atoi(c.DefaultQuery("page", "1"))
    perPage, _ := strconv.Atoi(c.DefaultQuery("per_page", "20"))

    users, total, err := h.service.List(page, perPage)
    if err != nil {
        response.Error(c, http.StatusInternalServerError, "INTERNAL_ERROR", "Failed to fetch users")
        return
    }

    // Convert to response DTOs
    userResponses := make([]*models.UserResponse, len(users))
    for i, u := range users {
        userResponses[i] = u.ToResponse()
    }

    totalPages := (total + perPage - 1) / perPage
    response.SuccessWithMeta(c, http.StatusOK, userResponses, &response.Meta{
        Page:       page,
        PerPage:    perPage,
        Total:      total,
        TotalPages: totalPages,
    })
}

func (h *UserHandler) Get(c *gin.Context) {
    id, err := strconv.ParseUint(c.Param("id"), 10, 32)
    if err != nil {
        response.Error(c, http.StatusBadRequest, "INVALID_ID", "Invalid user ID")
        return
    }

    user, err := h.service.GetByID(uint(id))
    if err != nil {
        if err == services.ErrUserNotFound {
            response.Error(c, http.StatusNotFound, "NOT_FOUND", "User not found")
            return
        }
        response.Error(c, http.StatusInternalServerError, "INTERNAL_ERROR", "Failed to fetch user")
        return
    }

    response.Success(c, http.StatusOK, user.ToResponse())
}

func (h *UserHandler) Update(c *gin.Context) {
    id, err := strconv.ParseUint(c.Param("id"), 10, 32)
    if err != nil {
        response.Error(c, http.StatusBadRequest, "INVALID_ID", "Invalid user ID")
        return
    }

    var req models.UpdateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.Error(c, http.StatusBadRequest, "VALIDATION_ERROR", err.Error())
        return
    }

    user, err := h.service.Update(uint(id), &req)
    if err != nil {
        if err == services.ErrUserNotFound {
            response.Error(c, http.StatusNotFound, "NOT_FOUND", "User not found")
            return
        }
        response.Error(c, http.StatusInternalServerError, "INTERNAL_ERROR", "Failed to update user")
        return
    }

    response.Success(c, http.StatusOK, user.ToResponse())
}

func (h *UserHandler) Delete(c *gin.Context) {
    id, err := strconv.ParseUint(c.Param("id"), 10, 32)
    if err != nil {
        response.Error(c, http.StatusBadRequest, "INVALID_ID", "Invalid user ID")
        return
    }

    if err := h.service.Delete(uint(id)); err != nil {
        if err == services.ErrUserNotFound {
            response.Error(c, http.StatusNotFound, "NOT_FOUND", "User not found")
            return
        }
        response.Error(c, http.StatusInternalServerError, "INTERNAL_ERROR", "Failed to delete user")
        return
    }

    c.Status(http.StatusNoContent)
}

func (h *UserHandler) GetCurrentUser(c *gin.Context) {
    // User ID set by auth middleware
    userID, exists := c.Get("userID")
    if !exists {
        response.Error(c, http.StatusUnauthorized, "UNAUTHORIZED", "User not authenticated")
        return
    }

    user, err := h.service.GetByID(userID.(uint))
    if err != nil {
        response.Error(c, http.StatusNotFound, "NOT_FOUND", "User not found")
        return
    }

    response.Success(c, http.StatusOK, user.ToResponse())
}
```

### Middleware

```go
// internal/middleware/auth.go
package middleware

import (
    "net/http"
    "strings"

    "github.com/gin-gonic/gin"
    "github.com/golang-jwt/jwt/v5"
    "myapp/pkg/response"
)

func Auth(jwtSecret string) gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            response.Error(c, http.StatusUnauthorized, "MISSING_TOKEN", "Authorization header required")
            c.Abort()
            return
        }

        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            response.Error(c, http.StatusUnauthorized, "INVALID_TOKEN", "Invalid authorization format")
            c.Abort()
            return
        }

        tokenString := parts[1]
        token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
            if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                return nil, jwt.ErrSignatureInvalid
            }
            return []byte(jwtSecret), nil
        })

        if err != nil || !token.Valid {
            response.Error(c, http.StatusUnauthorized, "INVALID_TOKEN", "Invalid or expired token")
            c.Abort()
            return
        }

        claims, ok := token.Claims.(jwt.MapClaims)
        if !ok {
            response.Error(c, http.StatusUnauthorized, "INVALID_TOKEN", "Invalid token claims")
            c.Abort()
            return
        }

        // Set user ID in context
        userID := uint(claims["user_id"].(float64))
        c.Set("userID", userID)

        c.Next()
    }
}
```

```go
// internal/middleware/cors.go
package middleware

import (
    "github.com/gin-gonic/gin"
)

func CORS() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("Access-Control-Allow-Origin", "*")
        c.Header("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
        c.Header("Access-Control-Allow-Headers", "Origin, Content-Type, Authorization")
        c.Header("Access-Control-Max-Age", "86400")

        if c.Request.Method == "OPTIONS" {
            c.AbortWithStatus(204)
            return
        }

        c.Next()
    }
}
```

```go
// internal/middleware/error.go
package middleware

import (
    "log"
    "net/http"

    "github.com/gin-gonic/gin"
    "myapp/pkg/response"
)

func ErrorHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()

        // Check for errors after handler execution
        if len(c.Errors) > 0 {
            err := c.Errors.Last()
            log.Printf("Error: %v", err.Err)

            // Don't expose internal errors
            response.Error(c, http.StatusInternalServerError, "INTERNAL_ERROR", "An error occurred")
        }
    }
}
```

### Service Layer

```go
// internal/services/user.go
package services

import (
    "errors"

    "golang.org/x/crypto/bcrypt"
    "myapp/internal/models"
    "myapp/internal/repository"
)

var (
    ErrUserNotFound = errors.New("user not found")
    ErrEmailExists  = errors.New("email already exists")
    ErrInvalidCreds = errors.New("invalid credentials")
)

type UserService struct {
    repo *repository.UserRepository
}

func NewUserService(repo *repository.UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) Create(req *models.CreateUserRequest) (*models.User, error) {
    // Check if email exists
    existing, _ := s.repo.FindByEmail(req.Email)
    if existing != nil {
        return nil, ErrEmailExists
    }

    // Hash password
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
    if err != nil {
        return nil, err
    }

    user := &models.User{
        Name:     req.Name,
        Email:    req.Email,
        Password: string(hashedPassword),
    }

    if err := s.repo.Create(user); err != nil {
        return nil, err
    }

    return user, nil
}

func (s *UserService) List(page, perPage int) ([]*models.User, int, error) {
    offset := (page - 1) * perPage
    return s.repo.FindAll(offset, perPage)
}

func (s *UserService) GetByID(id uint) (*models.User, error) {
    user, err := s.repo.FindByID(id)
    if err != nil {
        return nil, err
    }
    if user == nil {
        return nil, ErrUserNotFound
    }
    return user, nil
}

func (s *UserService) Update(id uint, req *models.UpdateUserRequest) (*models.User, error) {
    user, err := s.GetByID(id)
    if err != nil {
        return nil, err
    }

    if req.Name != "" {
        user.Name = req.Name
    }
    if req.Email != "" {
        user.Email = req.Email
    }

    if err := s.repo.Update(user); err != nil {
        return nil, err
    }

    return user, nil
}

func (s *UserService) Delete(id uint) error {
    user, err := s.GetByID(id)
    if err != nil {
        return err
    }
    return s.repo.Delete(user)
}

func (s *UserService) Authenticate(email, password string) (*models.User, error) {
    user, err := s.repo.FindByEmail(email)
    if err != nil || user == nil {
        return nil, ErrInvalidCreds
    }

    if err := bcrypt.CompareHashAndPassword([]byte(user.Password), []byte(password)); err != nil {
        return nil, ErrInvalidCreds
    }

    return user, nil
}
```

### Testing

```go
// internal/handlers/users_test.go
package handlers_test

import (
    "bytes"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/gin-gonic/gin"
    "github.com/stretchr/testify/assert"
    "myapp/internal/handlers"
    "myapp/internal/models"
)

func TestUserHandler_Register(t *testing.T) {
    gin.SetMode(gin.TestMode)

    // Setup mock service
    mockService := &MockUserService{}
    handler := handlers.NewUserHandler(mockService)

    router := gin.New()
    router.POST("/register", handler.Register)

    t.Run("successful registration", func(t *testing.T) {
        body := models.CreateUserRequest{
            Name:     "Test User",
            Email:    "test@example.com",
            Password: "password123",
        }
        jsonBody, _ := json.Marshal(body)

        req := httptest.NewRequest("POST", "/register", bytes.NewBuffer(jsonBody))
        req.Header.Set("Content-Type", "application/json")
        w := httptest.NewRecorder()

        router.ServeHTTP(w, req)

        assert.Equal(t, http.StatusCreated, w.Code)
    })

    t.Run("validation error", func(t *testing.T) {
        body := models.CreateUserRequest{
            Name:     "T", // Too short
            Email:    "invalid",
            Password: "short",
        }
        jsonBody, _ := json.Marshal(body)

        req := httptest.NewRequest("POST", "/register", bytes.NewBuffer(jsonBody))
        req.Header.Set("Content-Type", "application/json")
        w := httptest.NewRecorder()

        router.ServeHTTP(w, req)

        assert.Equal(t, http.StatusBadRequest, w.Code)
    })
}
```

---

## Anti-Patterns

### ❌ Using c.Bind* Instead of c.ShouldBind*

**Why it's bad**: `c.Bind*` aborts and writes response on error.

```go
// ❌ DON'T — Automatic abort hides errors
func Handler(c *gin.Context) {
    var req Request
    c.BindJSON(&req)  // Aborts on error, you lose control
    // ...
}

// ✅ DO — Explicit error handling
func Handler(c *gin.Context) {
    var req Request
    if err := c.ShouldBindJSON(&req); err != nil {
        response.Error(c, http.StatusBadRequest, "VALIDATION_ERROR", err.Error())
        return
    }
    // ...
}
```

### ❌ Exposing Internal Errors

**Why it's bad**: Leaks implementation details to attackers.

```go
// ❌ DON'T — Expose internal errors
func Handler(c *gin.Context) {
    user, err := db.FindUser(id)
    if err != nil {
        c.JSON(500, gin.H{"error": err.Error()})  // Leaks DB details!
        return
    }
}

// ✅ DO — Generic error messages
func Handler(c *gin.Context) {
    user, err := db.FindUser(id)
    if err != nil {
        log.Printf("Database error: %v", err)  // Log internally
        response.Error(c, 500, "INTERNAL_ERROR", "An error occurred")
        return
    }
}
```

### ❌ Not Using Gin Release Mode

**Why it's bad**: Debug mode has overhead and verbose output.

```go
// ❌ DON'T — Leave in debug mode
func main() {
    router := gin.Default()
    // ...
}

// ✅ DO — Set release mode in production
func main() {
    if os.Getenv("ENVIRONMENT") == "production" {
        gin.SetMode(gin.ReleaseMode)
    }
    router := gin.Default()
    // ...
}
```

### ❌ Not Implementing Graceful Shutdown

**Why it's bad**: In-flight requests are dropped.

```go
// ❌ DON'T — Direct listen
func main() {
    router.Run(":8080")  // No graceful shutdown
}

// ✅ DO — Graceful shutdown (see main.go example above)
```

### ❌ Middleware Order Mistakes

**Why it's bad**: Wrong order causes unexpected behavior.

```go
// ❌ DON'T — Auth before logger (no logging of auth failures)
router.Use(authMiddleware)
router.Use(gin.Logger())

// ✅ DO — Proper order
router.Use(gin.Logger())      // 1. Logging
router.Use(gin.Recovery())    // 2. Panic recovery
router.Use(corsMiddleware)    // 3. CORS
router.Use(authMiddleware)    // 4. Authentication (route-specific)
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 1.9 | 2023 | Stable release |
| 1.10 | 2024 | Performance improvements, bug fixes |
| 1.10.x | 2025 | Continued maintenance, security patches |

### Gin Ecosystem 2025

- **gin-contrib**: Official middleware collection
- **swaggo/gin-swagger**: OpenAPI documentation
- **go-playground/validator**: Advanced validation
- **gin-gonic/autotls**: Automatic HTTPS
- **Rate limiting**: gin-contrib/limiter

---

## Quick Reference

| Context Method | Purpose |
|----------------|---------|
| `c.Param("id")` | Get path parameter |
| `c.Query("page")` | Get query parameter |
| `c.DefaultQuery("page", "1")` | Query with default |
| `c.ShouldBindJSON(&obj)` | Parse JSON body |
| `c.ShouldBindQuery(&obj)` | Parse query params |
| `c.JSON(status, obj)` | Send JSON response |
| `c.Set("key", value)` | Set context value |
| `c.Get("key")` | Get context value |
| `c.Abort()` | Stop middleware chain |

| Validation Tag | Purpose |
|----------------|---------|
| `binding:"required"` | Field must be present |
| `binding:"email"` | Must be valid email |
| `binding:"min=2,max=100"` | Length constraints |
| `binding:"oneof=a b c"` | Must be one of values |
| `binding:"omitempty"` | Skip if empty |
| `binding:"dive"` | Validate slice elements |

| Middleware | Purpose |
|------------|---------|
| `gin.Logger()` | Request logging |
| `gin.Recovery()` | Panic recovery |
| `cors.Default()` | CORS handling |
| `gzip.Gzip()` | Response compression |
| `limiter.Limiter()` | Rate limiting |

| Command | Purpose |
|---------|---------|
| `go run cmd/server/main.go` | Start server |
| `go test ./...` | Run all tests |
| `go build -o app cmd/server/main.go` | Build binary |
| `air` | Hot reload (install air) |

---

## When to Use Gin

| Great For | Not Ideal For |
|-----------|---------------|
| REST APIs | GraphQL (use gqlgen) |
| Microservices | Full-stack web apps |
| High-performance APIs | Simple scripts |
| Teams familiar with Go | Rapid prototypes |
| Production workloads | Learning web dev |

---

## Resources

- [Official Gin Documentation](https://gin-gonic.com/docs/)
- [Gin GitHub](https://github.com/gin-gonic/gin)
- [Go REST API Tutorial](https://go.dev/doc/tutorial/web-service-gin)
- [Gin Best Practices 2025](https://oneuptime.com/blog/post/2026-01-07-go-rest-api-gin/view)
- [Go Web Frameworks Comparison](https://www.jhkinfotech.com/blog/golang-web-framework)
- [Gin Error Handling](https://leapcell.medium.com/robust-error-handling-in-go-web-projects-with-gin-58eba3b06e6e)
