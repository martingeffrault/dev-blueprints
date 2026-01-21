# FastAPI (2025)

> **Last updated**: January 2026
> **Versions covered**: 0.110+
> **Purpose**: Modern, fast Python web framework for building APIs

---

## Philosophy (2025-2026)

FastAPI is the **async-first, type-safe API framework** for Python, combining speed with automatic OpenAPI documentation.

**Key philosophical shifts:**
- **Async by design** — Built on Starlette and Pydantic
- **Type hints = validation** — Pydantic v2 for runtime validation
- **3000+ req/sec** — Performance rivaling Node.js and Go
- **OpenAPI automatic** — Documentation generated from code
- **Dependency injection** — First-class DI support
- **Production mainstream** — Standard for ML serving, data platforms

---

## TL;DR

- Use `async def` only for I/O-bound operations with async libraries
- Use `def` for sync operations — FastAPI runs them in threadpool
- Never block the event loop with sync calls in `async def`
- Use `asyncio.gather()` for concurrent independent operations
- Use connection pooling for databases
- CPU-bound work → Celery/multiprocessing, not async
- One endpoint should never call another endpoint

---

## Best Practices

### Project Structure

```
my_api/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI app instance
│   ├── config.py            # Settings with pydantic-settings
│   ├── dependencies.py      # Shared dependencies
│   ├── database.py          # Database connection
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── users.py
│   │   ├── posts.py
│   │   └── auth.py
│   ├── models/              # SQLAlchemy/Pydantic models
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── post.py
│   ├── schemas/             # Pydantic schemas
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── post.py
│   ├── services/            # Business logic
│   │   ├── __init__.py
│   │   ├── user_service.py
│   │   └── post_service.py
│   ├── repositories/        # Data access layer
│   │   ├── __init__.py
│   │   └── user_repository.py
│   └── utils/
│       └── security.py
├── tests/
├── alembic/                 # Database migrations
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

### App Configuration

```python
# app/config.py
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    app_name: str = "My API"
    debug: bool = False
    database_url: str
    secret_key: str
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30

    class Config:
        env_file = ".env"

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

```python
# app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager
from app.config import get_settings
from app.database import engine
from app.routers import users, posts, auth

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    await engine.connect()
    yield
    # Shutdown
    await engine.dispose()

app = FastAPI(
    title=get_settings().app_name,
    lifespan=lifespan,
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(auth.router, prefix="/auth", tags=["auth"])
app.include_router(users.router, prefix="/users", tags=["users"])
app.include_router(posts.router, prefix="/posts", tags=["posts"])

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

### Async Database (SQLAlchemy 2.0)

```python
# app/database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker, declarative_base
from app.config import get_settings

settings = get_settings()

engine = create_async_engine(
    settings.database_url,
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,
    pool_recycle=3600,
)

AsyncSessionLocal = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)

Base = declarative_base()

async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

### Pydantic Schemas

```python
# app/schemas/user.py
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime

class UserBase(BaseModel):
    email: EmailStr
    name: str = Field(min_length=2, max_length=100)

class UserCreate(UserBase):
    password: str = Field(min_length=8)

class UserUpdate(BaseModel):
    name: str | None = Field(default=None, min_length=2, max_length=100)
    email: EmailStr | None = None

class UserResponse(UserBase):
    id: int
    created_at: datetime
    is_active: bool

    model_config = {"from_attributes": True}

class UserList(BaseModel):
    items: list[UserResponse]
    total: int
    page: int
    per_page: int
```

### Router with Dependencies

```python
# app/routers/users.py
from fastapi import APIRouter, Depends, HTTPException, status, Query
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import get_db
from app.schemas.user import UserCreate, UserResponse, UserUpdate, UserList
from app.services.user_service import UserService
from app.dependencies import get_current_user

router = APIRouter()

@router.post("/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(
    user_data: UserCreate,
    db: AsyncSession = Depends(get_db),
):
    """Create a new user."""
    service = UserService(db)
    user = await service.create_user(user_data)
    return user

@router.get("/", response_model=UserList)
async def list_users(
    page: int = Query(default=1, ge=1),
    per_page: int = Query(default=20, ge=1, le=100),
    db: AsyncSession = Depends(get_db),
):
    """List all users with pagination."""
    service = UserService(db)
    users, total = await service.list_users(page=page, per_page=per_page)
    return UserList(items=users, total=total, page=page, per_page=per_page)

@router.get("/me", response_model=UserResponse)
async def get_current_user_info(
    current_user = Depends(get_current_user),
):
    """Get current authenticated user."""
    return current_user

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),
):
    """Get user by ID."""
    service = UserService(db)
    user = await service.get_user(user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )
    return user
```

### Service Layer

```python
# app/services/user_service.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func
from app.models.user import User
from app.schemas.user import UserCreate, UserUpdate
from app.utils.security import hash_password

class UserService:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def create_user(self, data: UserCreate) -> User:
        """Create a new user."""
        # Check if email exists
        existing = await self.get_user_by_email(data.email)
        if existing:
            raise ValueError("Email already registered")

        user = User(
            email=data.email,
            name=data.name,
            hashed_password=hash_password(data.password),
        )
        self.db.add(user)
        await self.db.flush()
        await self.db.refresh(user)
        return user

    async def get_user(self, user_id: int) -> User | None:
        """Get user by ID."""
        result = await self.db.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()

    async def get_user_by_email(self, email: str) -> User | None:
        """Get user by email."""
        result = await self.db.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()

    async def list_users(
        self, page: int = 1, per_page: int = 20
    ) -> tuple[list[User], int]:
        """List users with pagination."""
        offset = (page - 1) * per_page

        # Get total count
        count_result = await self.db.execute(select(func.count(User.id)))
        total = count_result.scalar()

        # Get paginated users
        result = await self.db.execute(
            select(User)
            .order_by(User.created_at.desc())
            .offset(offset)
            .limit(per_page)
        )
        users = result.scalars().all()

        return users, total
```

### Concurrent Async Operations

```python
# ✅ Run independent operations concurrently
import asyncio

@router.get("/dashboard")
async def get_dashboard(
    db: AsyncSession = Depends(get_db),
    current_user = Depends(get_current_user),
):
    # Run all queries concurrently
    user_stats, recent_posts, notifications = await asyncio.gather(
        get_user_stats(db, current_user.id),
        get_recent_posts(db, current_user.id),
        get_notifications(db, current_user.id),
    )
    return {
        "stats": user_stats,
        "posts": recent_posts,
        "notifications": notifications,
    }
```

### Background Tasks

```python
from fastapi import BackgroundTasks

@router.post("/users/")
async def create_user(
    user_data: UserCreate,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db),
):
    user = await user_service.create_user(db, user_data)
    # Run email sending in background
    background_tasks.add_task(send_welcome_email, user.email)
    return user
```

### Dependency Injection

```python
# app/dependencies.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt
from app.config import get_settings
from app.database import get_db
from app.services.user_service import UserService

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db = Depends(get_db),
):
    """Get current authenticated user from JWT token."""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        settings = get_settings()
        payload = jwt.decode(token, settings.secret_key, algorithms=[settings.algorithm])
        user_id: int = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    service = UserService(db)
    user = await service.get_user(user_id)
    if user is None:
        raise credentials_exception
    return user
```

---

## Anti-Patterns

### ❌ Blocking I/O in async def

**Why it's bad**: Blocks the entire event loop.

```python
# ❌ DON'T — Sync call in async function
@app.get("/users")
async def get_users():
    users = requests.get("http://api/users")  # Blocks event loop!
    return users.json()

# ✅ DO — Use async HTTP client
import httpx

@app.get("/users")
async def get_users():
    async with httpx.AsyncClient() as client:
        response = await client.get("http://api/users")
        return response.json()
```

### ❌ Sequential await Instead of Concurrent

**Why it's bad**: 3 sequential 1-second calls = 3 seconds.

```python
# ❌ DON'T — Sequential (slow)
@app.get("/dashboard")
async def dashboard():
    users = await get_users()      # 1 sec
    posts = await get_posts()      # 1 sec
    stats = await get_stats()      # 1 sec
    # Total: 3 seconds

# ✅ DO — Concurrent
@app.get("/dashboard")
async def dashboard():
    users, posts, stats = await asyncio.gather(
        get_users(),
        get_posts(),
        get_stats(),
    )
    # Total: 1 second
```

### ❌ Endpoint Calling Another Endpoint

**Why it's bad**: Extra serialization, auth, latency.

```python
# ❌ DON'T
@app.get("/user-summary/{user_id}")
async def get_user_summary(user_id: int):
    user_response = await client.get(f"/users/{user_id}")  # HTTP call to self!
    return {"summary": process(user_response)}

# ✅ DO — Call service directly
@app.get("/user-summary/{user_id}")
async def get_user_summary(user_id: int, db = Depends(get_db)):
    user = await user_service.get_user(db, user_id)
    return {"summary": process(user)}
```

### ❌ Not Using Connection Pooling

**Why it's bad**: Connection exhaustion under load.

```python
# ❌ DON'T — New connection per request
async def get_db():
    engine = create_async_engine(DATABASE_URL)
    async with AsyncSession(engine) as session:
        yield session

# ✅ DO — Reuse connection pool (see database setup above)
```

### ❌ CPU-Bound Work in Async

**Why it's bad**: Blocks event loop, no benefit from async.

```python
# ❌ DON'T
@app.post("/process")
async def process_data(data: LargeData):
    result = heavy_cpu_computation(data)  # Blocks!
    return result

# ✅ DO — Offload to process pool
from concurrent.futures import ProcessPoolExecutor
import asyncio

executor = ProcessPoolExecutor()

@app.post("/process")
async def process_data(data: LargeData):
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(executor, heavy_cpu_computation, data)
    return result

# ✅ BETTER — Use Celery for long-running tasks
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 0.100+ | 2024 | Pydantic v2 full support |
| 0.110+ | 2025 | Performance improvements |
| 2025 | Trend | Mainstream for ML/data APIs |

---

## Quick Reference

| Task | Solution |
|------|----------|
| Run dev server | `uvicorn app.main:app --reload` |
| Run production | `gunicorn app.main:app -w 4 -k uvicorn.workers.UvicornWorker` |
| Auto-generated docs | `/docs` (Swagger) or `/redoc` |
| Path parameter | `@app.get("/users/{user_id}")` |
| Query parameter | `def func(skip: int = 0, limit: int = 10)` |
| Request body | `def func(user: UserCreate)` |
| Dependency | `Depends(get_db)` |
| Background task | `BackgroundTasks.add_task()` |

| When to Use | async def vs def |
|-------------|------------------|
| Async I/O (httpx, asyncpg) | `async def` |
| Sync I/O (requests, psycopg2) | `def` |
| CPU-bound work | `def` + ProcessPool |
| No I/O | Either works |

---

## Resources

- [Official FastAPI Documentation](https://fastapi.tiangolo.com/)
- [FastAPI Best Practices](https://github.com/zhanymkanov/fastapi-best-practices)
- [10 Async Pitfalls in FastAPI](https://medium.com/@bhagyarana80/10-async-pitfalls-in-fastapi-and-how-to-avoid-them-60d6c67ea48f)
- [FastAPI Mistakes That Kill Performance](https://dev.to/igorbenav/fastapi-mistakes-that-kill-your-performance-2b8k)
- [Async APIs with FastAPI](https://shiladityamajumder.medium.com/async-apis-with-fastapi-patterns-pitfalls-best-practices-2d72b2b66f25)
- [FastAPI Production Deployment](https://render.com/articles/fastapi-production-deployment-best-practices)
