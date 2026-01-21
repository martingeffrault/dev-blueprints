# Axum (2025)

> **Last updated**: January 2026
> **Versions covered**: Axum 0.7–0.8+
> **Purpose**: Ergonomic, modular Rust web framework built on Tokio and Tower

---

## Philosophy (2025-2026)

Axum is the **ergonomic Rust web framework** built by the Tokio team. It embraces a lean design — providing core abstractions while leveraging Tower's middleware ecosystem for everything else.

**Key philosophical shifts:**
- **Macro-free API** — No magic, just types
- **Tower middleware** — Reuse existing ecosystem
- **Extractor pattern** — Type-safe request parsing
- **Composable** — Small, focused components
- **Performance** — Minimal overhead over Hyper
- **Error handling** — Infallible handlers, custom error types
- **Async-first** — Built on Tokio runtime

---

## TL;DR

- Use extractors for type-safe request parsing
- Wrap state in `Arc` for thread-safe sharing
- Create custom error types implementing `IntoResponse`
- Use Tower middleware (timeout, tracing, compression)
- Path params use `/{param}` syntax (not `/:param`)
- Use `#[debug_handler]` macro for better error messages
- Use `spawn_blocking` for CPU-intensive tasks
- Never use `unwrap()` in handlers — return `Result`

---

## Best Practices

### Project Structure

```
my-axum-app/
├── src/
│   ├── main.rs
│   ├── lib.rs
│   ├── config.rs
│   ├── routes/
│   │   ├── mod.rs
│   │   ├── users.rs
│   │   └── health.rs
│   ├── handlers/
│   │   ├── mod.rs
│   │   └── users.rs
│   ├── models/
│   │   ├── mod.rs
│   │   └── user.rs
│   ├── services/
│   │   ├── mod.rs
│   │   └── user_service.rs
│   ├── db/
│   │   ├── mod.rs
│   │   └── postgres.rs
│   ├── error.rs
│   └── middleware/
│       ├── mod.rs
│       └── auth.rs
├── migrations/
├── tests/
│   └── integration/
├── Cargo.toml
└── .env
```

### Basic Server Setup

```rust
// src/main.rs
use axum::{routing::get, Router};
use std::net::SocketAddr;
use tokio::net::TcpListener;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

mod config;
mod error;
mod handlers;
mod models;
mod routes;
mod services;

#[tokio::main]
async fn main() {
    // Initialize tracing
    tracing_subscriber::registry()
        .with(tracing_subscriber::fmt::layer())
        .with(tracing_subscriber::EnvFilter::from_default_env())
        .init();

    // Load configuration
    let config = config::Config::from_env();

    // Build application
    let app = routes::create_router(config).await;

    // Start server
    let addr = SocketAddr::from(([0, 0, 0, 0], 3000));
    let listener = TcpListener::bind(addr).await.unwrap();

    tracing::info!("Server running on {}", addr);

    axum::serve(listener, app).await.unwrap();
}
```

### Router Setup

```rust
// src/routes/mod.rs
use axum::{
    routing::{get, post, put, delete},
    Router,
};
use std::sync::Arc;
use tower_http::{
    compression::CompressionLayer,
    cors::{Any, CorsLayer},
    trace::TraceLayer,
};

use crate::{config::Config, handlers, services::UserService};

pub struct AppState {
    pub user_service: UserService,
    pub config: Config,
}

pub async fn create_router(config: Config) -> Router {
    // Initialize services
    let pool = sqlx::PgPool::connect(&config.database_url)
        .await
        .expect("Failed to connect to database");

    let user_service = UserService::new(pool);

    let state = Arc::new(AppState {
        user_service,
        config,
    });

    Router::new()
        // Health check
        .route("/health", get(handlers::health::check))
        // User routes
        .route("/api/users", get(handlers::users::list))
        .route("/api/users", post(handlers::users::create))
        .route("/api/users/{id}", get(handlers::users::get))
        .route("/api/users/{id}", put(handlers::users::update))
        .route("/api/users/{id}", delete(handlers::users::delete))
        // Middleware
        .layer(TraceLayer::new_for_http())
        .layer(CompressionLayer::new())
        .layer(
            CorsLayer::new()
                .allow_origin(Any)
                .allow_methods(Any)
                .allow_headers(Any),
        )
        .with_state(state)
}
```

### Handlers with Extractors

```rust
// src/handlers/users.rs
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    Json,
};
use serde::{Deserialize, Serialize};
use std::sync::Arc;

use crate::{
    error::AppError,
    models::user::{CreateUser, UpdateUser, User},
    routes::AppState,
};

// List users with pagination
#[derive(Deserialize)]
pub struct ListParams {
    #[serde(default = "default_page")]
    page: u32,
    #[serde(default = "default_limit")]
    limit: u32,
}

fn default_page() -> u32 { 1 }
fn default_limit() -> u32 { 20 }

pub async fn list(
    State(state): State<Arc<AppState>>,
    Query(params): Query<ListParams>,
) -> Result<Json<Vec<User>>, AppError> {
    let users = state.user_service
        .list(params.page, params.limit)
        .await?;

    Ok(Json(users))
}

// Get single user
pub async fn get(
    State(state): State<Arc<AppState>>,
    Path(id): Path<i32>,
) -> Result<Json<User>, AppError> {
    let user = state.user_service
        .get_by_id(id)
        .await?
        .ok_or(AppError::NotFound)?;

    Ok(Json(user))
}

// Create user
pub async fn create(
    State(state): State<Arc<AppState>>,
    Json(input): Json<CreateUser>,
) -> Result<(StatusCode, Json<User>), AppError> {
    let user = state.user_service.create(input).await?;
    Ok((StatusCode::CREATED, Json(user)))
}

// Update user
pub async fn update(
    State(state): State<Arc<AppState>>,
    Path(id): Path<i32>,
    Json(input): Json<UpdateUser>,
) -> Result<Json<User>, AppError> {
    let user = state.user_service
        .update(id, input)
        .await?
        .ok_or(AppError::NotFound)?;

    Ok(Json(user))
}

// Delete user
pub async fn delete(
    State(state): State<Arc<AppState>>,
    Path(id): Path<i32>,
) -> Result<StatusCode, AppError> {
    state.user_service.delete(id).await?;
    Ok(StatusCode::NO_CONTENT)
}
```

### Custom Error Handling

```rust
// src/error.rs
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;

#[derive(Debug)]
pub enum AppError {
    NotFound,
    BadRequest(String),
    Unauthorized,
    Forbidden,
    InternalError(String),
    ValidationError(Vec<String>),
    DatabaseError(sqlx::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, error_message) = match self {
            AppError::NotFound => (
                StatusCode::NOT_FOUND,
                "Resource not found".to_string(),
            ),
            AppError::BadRequest(msg) => (
                StatusCode::BAD_REQUEST,
                msg,
            ),
            AppError::Unauthorized => (
                StatusCode::UNAUTHORIZED,
                "Unauthorized".to_string(),
            ),
            AppError::Forbidden => (
                StatusCode::FORBIDDEN,
                "Forbidden".to_string(),
            ),
            AppError::InternalError(msg) => {
                tracing::error!("Internal error: {}", msg);
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "Internal server error".to_string(),
                )
            }
            AppError::ValidationError(errors) => (
                StatusCode::UNPROCESSABLE_ENTITY,
                errors.join(", "),
            ),
            AppError::DatabaseError(err) => {
                tracing::error!("Database error: {:?}", err);
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "Database error".to_string(),
                )
            }
        };

        let body = Json(json!({
            "error": error_message,
        }));

        (status, body).into_response()
    }
}

// Implement From for automatic conversion
impl From<sqlx::Error> for AppError {
    fn from(err: sqlx::Error) -> Self {
        AppError::DatabaseError(err)
    }
}
```

### Models with Validation

```rust
// src/models/user.rs
use serde::{Deserialize, Serialize};
use validator::Validate;

#[derive(Debug, Serialize, sqlx::FromRow)]
pub struct User {
    pub id: i32,
    pub name: String,
    pub email: String,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

#[derive(Debug, Deserialize, Validate)]
pub struct CreateUser {
    #[validate(length(min = 2, max = 100))]
    pub name: String,

    #[validate(email)]
    pub email: String,

    #[validate(length(min = 8))]
    pub password: String,
}

#[derive(Debug, Deserialize, Validate)]
pub struct UpdateUser {
    #[validate(length(min = 2, max = 100))]
    pub name: Option<String>,

    #[validate(email)]
    pub email: Option<String>,
}
```

### Validation Extractor

```rust
// src/handlers/mod.rs
use axum::{
    async_trait,
    extract::{FromRequest, Request},
    http::StatusCode,
    Json,
};
use serde::de::DeserializeOwned;
use validator::Validate;

use crate::error::AppError;

// Custom extractor that validates input
pub struct ValidatedJson<T>(pub T);

#[async_trait]
impl<S, T> FromRequest<S> for ValidatedJson<T>
where
    S: Send + Sync,
    T: DeserializeOwned + Validate,
    Json<T>: FromRequest<S>,
{
    type Rejection = AppError;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|_| AppError::BadRequest("Invalid JSON".to_string()))?;

        value.validate().map_err(|e| {
            let errors: Vec<String> = e
                .field_errors()
                .into_iter()
                .flat_map(|(field, errors)| {
                    errors.iter().map(move |e| {
                        format!("{}: {}", field, e.message.as_ref().unwrap_or(&"invalid".into()))
                    })
                })
                .collect();
            AppError::ValidationError(errors)
        })?;

        Ok(ValidatedJson(value))
    }
}

// Usage in handler
pub async fn create(
    State(state): State<Arc<AppState>>,
    ValidatedJson(input): ValidatedJson<CreateUser>,
) -> Result<(StatusCode, Json<User>), AppError> {
    // input is already validated
    let user = state.user_service.create(input).await?;
    Ok((StatusCode::CREATED, Json(user)))
}
```

### Service Layer

```rust
// src/services/user_service.rs
use sqlx::PgPool;

use crate::models::user::{CreateUser, UpdateUser, User};

#[derive(Clone)]
pub struct UserService {
    pool: PgPool,
}

impl UserService {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }

    pub async fn list(&self, page: u32, limit: u32) -> Result<Vec<User>, sqlx::Error> {
        let offset = (page - 1) * limit;

        sqlx::query_as::<_, User>(
            "SELECT id, name, email, created_at FROM users
             ORDER BY created_at DESC
             LIMIT $1 OFFSET $2"
        )
        .bind(limit as i64)
        .bind(offset as i64)
        .fetch_all(&self.pool)
        .await
    }

    pub async fn get_by_id(&self, id: i32) -> Result<Option<User>, sqlx::Error> {
        sqlx::query_as::<_, User>(
            "SELECT id, name, email, created_at FROM users WHERE id = $1"
        )
        .bind(id)
        .fetch_optional(&self.pool)
        .await
    }

    pub async fn create(&self, input: CreateUser) -> Result<User, sqlx::Error> {
        let password_hash = hash_password(&input.password);

        sqlx::query_as::<_, User>(
            "INSERT INTO users (name, email, password_hash)
             VALUES ($1, $2, $3)
             RETURNING id, name, email, created_at"
        )
        .bind(&input.name)
        .bind(&input.email)
        .bind(&password_hash)
        .fetch_one(&self.pool)
        .await
    }

    pub async fn update(&self, id: i32, input: UpdateUser) -> Result<Option<User>, sqlx::Error> {
        sqlx::query_as::<_, User>(
            "UPDATE users
             SET name = COALESCE($2, name),
                 email = COALESCE($3, email)
             WHERE id = $1
             RETURNING id, name, email, created_at"
        )
        .bind(id)
        .bind(&input.name)
        .bind(&input.email)
        .fetch_optional(&self.pool)
        .await
    }

    pub async fn delete(&self, id: i32) -> Result<bool, sqlx::Error> {
        let result = sqlx::query("DELETE FROM users WHERE id = $1")
            .bind(id)
            .execute(&self.pool)
            .await?;

        Ok(result.rows_affected() > 0)
    }
}

fn hash_password(password: &str) -> String {
    // Use argon2 or bcrypt in production
    format!("hashed_{}", password)
}
```

### Authentication Middleware

```rust
// src/middleware/auth.rs
use axum::{
    extract::Request,
    http::{header, StatusCode},
    middleware::Next,
    response::Response,
};

use crate::error::AppError;

pub async fn auth_middleware(
    request: Request,
    next: Next,
) -> Result<Response, AppError> {
    let auth_header = request
        .headers()
        .get(header::AUTHORIZATION)
        .and_then(|h| h.to_str().ok());

    match auth_header {
        Some(header) if header.starts_with("Bearer ") => {
            let token = &header[7..];

            // Validate token
            if validate_token(token).await.is_ok() {
                Ok(next.run(request).await)
            } else {
                Err(AppError::Unauthorized)
            }
        }
        _ => Err(AppError::Unauthorized),
    }
}

async fn validate_token(token: &str) -> Result<(), ()> {
    // Validate JWT or session token
    if token.is_empty() {
        return Err(());
    }
    Ok(())
}

// Apply to specific routes
pub fn protected_routes(state: Arc<AppState>) -> Router {
    Router::new()
        .route("/api/admin/users", get(handlers::admin::list_users))
        .route_layer(axum::middleware::from_fn(auth_middleware))
        .with_state(state)
}
```

### Testing

```rust
// tests/integration/users.rs
use axum::{
    body::Body,
    http::{Request, StatusCode},
};
use tower::ServiceExt;
use serde_json::json;

use my_app::routes::create_router;

#[tokio::test]
async fn test_create_user() {
    let app = create_router(test_config()).await;

    let response = app
        .oneshot(
            Request::builder()
                .method("POST")
                .uri("/api/users")
                .header("Content-Type", "application/json")
                .body(Body::from(
                    json!({
                        "name": "Test User",
                        "email": "test@example.com",
                        "password": "password123"
                    })
                    .to_string(),
                ))
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::CREATED);
}

#[tokio::test]
async fn test_get_user_not_found() {
    let app = create_router(test_config()).await;

    let response = app
        .oneshot(
            Request::builder()
                .uri("/api/users/99999")
                .body(Body::empty())
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::NOT_FOUND);
}
```

---

## Anti-Patterns

### ❌ Using unwrap() in Handlers

**Why it's bad**: Panics crash the handler; returns empty response.

```rust
// ❌ DON'T — Panic in handlers
pub async fn get_user(Path(id): Path<i32>) -> Json<User> {
    let user = db.get_user(id).await.unwrap();  // Panics on error!
    Json(user)
}

// ✅ DO — Return Result with proper error
pub async fn get_user(
    Path(id): Path<i32>,
) -> Result<Json<User>, AppError> {
    let user = db.get_user(id).await?
        .ok_or(AppError::NotFound)?;
    Ok(Json(user))
}
```

### ❌ Blocking the Async Runtime

**Why it's bad**: Blocks all concurrent requests.

```rust
// ❌ DON'T — CPU-intensive work in async
pub async fn process(data: String) -> String {
    // This blocks the entire runtime!
    expensive_computation(&data)
}

// ✅ DO — Use spawn_blocking
pub async fn process(data: String) -> Result<String, AppError> {
    tokio::task::spawn_blocking(move || {
        expensive_computation(&data)
    })
    .await
    .map_err(|e| AppError::InternalError(e.to_string()))
}
```

### ❌ Not Sharing State Properly

**Why it's bad**: Each request gets a clone, mutations lost.

```rust
// ❌ DON'T — State without Arc
#[derive(Clone)]
struct AppState {
    counter: usize,  // Each clone is independent!
}

// ✅ DO — Wrap in Arc for sharing
struct AppState {
    counter: Arc<AtomicUsize>,
    db: PgPool,  // PgPool is already Arc internally
}

let state = Arc::new(AppState { ... });
router.with_state(state)
```

### ❌ Old Path Parameter Syntax

**Why it's bad**: Changed in Axum 0.8.

```rust
// ❌ DON'T — Old syntax
.route("/users/:id", get(get_user))

// ✅ DO — New syntax (0.8+)
.route("/users/{id}", get(get_user))
```

### ❌ Mixing Sync and Async I/O

**Why it's bad**: Sync I/O blocks async runtime.

```rust
// ❌ DON'T — Sync file I/O
pub async fn read_file() -> String {
    std::fs::read_to_string("file.txt").unwrap()  // Blocks!
}

// ✅ DO — Async file I/O
pub async fn read_file() -> Result<String, AppError> {
    tokio::fs::read_to_string("file.txt")
        .await
        .map_err(|e| AppError::InternalError(e.to_string()))
}
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 0.7 | 2024 | Stable foundation |
| **0.8.0** | **Jan 1, 2025** | **Path syntax change** `/{param}`, `Option<T>` extractor change, new serve API |
| 0.8.x | 2025 | Latest stable, 22k+ GitHub stars, community favorite |
| 0.9 | In Progress | Breaking changes on main branch |

### Axum 0.8.0 Breaking Changes (January 2025)

**Path Parameter Syntax Changed**
```rust
// ❌ Old syntax (0.7 and earlier)
.route("/users/:id", get(get_user))
.route("/files/*path", get(get_file))

// ✅ New syntax (0.8+)
.route("/users/{id}", get(get_user))
.route("/files/{*path}", get(get_file))

// Escape literal braces with double braces
.route("/json/{{key}}", get(literal_braces))  // matches "/json/{key}"
```

**Why the change:**
- Old syntax didn't allow route definitions with leading `:` or `*` characters
- New syntax matches `format!()` macro style
- Aligns with OpenAPI path parameter format

**Option<T> Extractor Behavior Changed**
```rust
// Before 0.8: Any rejection from T was silently turned into None

// After 0.8: Option<T> requires T to implement OptionalFromRequestParts
// This allows proper rejection handling while still being optional

use axum::extract::OptionalPath;

// For optional path parameters, use new dedicated extractor
pub async fn handler(OptionalPath(id): OptionalPath<i32>) -> impl IntoResponse {
    match id {
        Some(id) => format!("ID: {}", id),
        None => "No ID provided".to_string(),
    }
}
```

**axum-extra Feature Flags Now Required**
```toml
# Cargo.toml - features now require explicit opt-in
[dependencies]
axum-extra = { version = "0.10", features = [
    "cached",           # Cached extractor
    "handler",          # Handler utilities
    "middleware",       # Middleware utilities
    "optional-path",    # OptionalPath extractor
    "routing",          # Routing utilities
    "with-rejection",   # WithRejection extractor
] }
```

**prost Dependency Upgraded**
- prost upgraded to v0.14 for protobuf support

**option_layer Response Body Type**
- `option_layer` now maps Response body type to `axum::body::Body`

### Axum Framework Position (2025-2026)

- **Surpassed Rocket** as community favorite for new Rust projects
- 22k+ GitHub stars — far ahead of other frameworks from same era
- Tight Tokio ecosystem integration
- Predicted: Full-stack Rust development with Leptos integration by mid-2025

### What's Coming in 0.9

- Currently on main branch (breaking changes)
- Tighter Leptos framework integration
- Enhanced server-side rendering capabilities
- More beginner-friendly documentation

---

## Quick Reference

| Extractor | Purpose |
|-----------|---------|
| `Path<T>` | URL path parameters |
| `Query<T>` | Query string parameters |
| `Json<T>` | JSON request body |
| `State<T>` | Shared application state |
| `Extension<T>` | Request extensions |
| `Headers` | Request headers |
| `Form<T>` | Form data |

| Response Type | Usage |
|---------------|-------|
| `Json<T>` | JSON response |
| `Html<String>` | HTML response |
| `StatusCode` | Status only |
| `(StatusCode, T)` | Status + body |
| `Response` | Full control |
| `Result<T, E>` | Success or error |

| Middleware | Purpose |
|------------|---------|
| `TraceLayer` | Request tracing |
| `CompressionLayer` | Response compression |
| `CorsLayer` | CORS headers |
| `TimeoutLayer` | Request timeout |
| `RateLimitLayer` | Rate limiting |

| Command | Purpose |
|---------|---------|
| `cargo run` | Start server |
| `cargo watch -x run` | Hot reload |
| `cargo test` | Run tests |
| `cargo clippy` | Lint code |

---

## When to Use Axum

| Great For | Not Ideal For |
|-----------|---------------|
| High-performance APIs | Quick prototypes |
| Microservices | Teams new to Rust |
| CPU-intensive workloads | Simple CRUD apps |
| Type-safe APIs | Dynamic typing needed |
| Tower middleware reuse | Rapid iteration |

---

## Resources

- [Official Axum Documentation](https://docs.rs/axum/latest/axum/)
- [Axum GitHub](https://github.com/tokio-rs/axum)
- [Complete Axum Guide 2025](https://www.shuttle.dev/blog/2023/12/06/using-axum-rust)
- [Axum Error Handling](https://blog.logrocket.com/rust-axum-error-handling/)
- [Building Clean Rust Backend with Axum](https://medium.com/@qkpiot/building-a-robust-rust-backend-with-axum-diesel-postgresql-and-ddd-from-concept-to-deployment-b25cf5c65bc8)
- [Rust Web Frameworks Benchmark 2025](https://markaicode.com/rust-web-frameworks-performance-benchmark-2025/)
