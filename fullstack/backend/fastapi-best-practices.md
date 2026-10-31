# FastAPI Best Practices & Patterns

A comprehensive guide to building production-ready FastAPI applications with proper structure, validation, error handling, and documentation.

## Table of Contents

1. [Project Structure](#project-structure)
2. [Endpoint Design Patterns](#endpoint-design-patterns)
3. [Request Validation with Pydantic](#request-validation-with-pydantic)
4. [Response Standardization](#response-standardization)
5. [Error Handling](#error-handling)
6. [Database Integration](#database-integration)
7. [CORS Configuration](#cors-configuration)
8. [Async Patterns](#async-patterns)
9. [API Documentation](#api-documentation)
10. [Performance Optimization](#performance-optimization)

---

## Project Structure

Organize your FastAPI project with clear separation of concerns:

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#1f77b4','primaryTextColor':'#fff','primaryBorderColor':'#145a8c','lineColor':'#5a5a5a','secondaryColor':'#ff7f0e','tertiaryColor':'#2ca02c'}}}%%
graph TD
    ROOT[my-api/]:::blue --> SRC[src/]:::green
    ROOT --> TESTS[tests/]:::orange
    ROOT --> ENV[.env]:::gray
    ROOT --> REQ[requirements.txt]:::gray

    SRC --> MAIN[main.py]:::green
    SRC --> CORE[core/]:::cyan
    SRC --> MOD[models/]:::purple
    SRC --> SCH[schemas/]:::teal
    SRC --> API[api/]:::orange
    SRC --> SRV[services/]:::blue

    CORE --> DB[database.py]:::cyan
    CORE --> CONF[config.py]:::cyan
    CORE --> SEC[security.py]:::cyan

    classDef blue fill:#4A90E2,stroke:#2E5C8A,color:#fff
    classDef green fill:#50C878,stroke:#2E8B57,color:#fff
    classDef purple fill:#9B59B6,stroke:#6C3483,color:#fff
    classDef orange fill:#F39C12,stroke:#BA7A0A,color:#fff
    classDef cyan fill:#3498DB,stroke:#2471A3,color:#fff
    classDef teal fill:#1ABC9C,stroke:#138D75,color:#fff
    classDef gray fill:#95A5A6,stroke:#7F8C8D,color:#333
```

**Key principles:**
- **`main.py`** - Application entry point only
- **`models/`** - SQLAlchemy database models
- **`schemas/`** - Pydantic validation models (separate from DB models!)
- **`api/`** - Endpoint definitions
- **`services/`** - Business logic (keep endpoints thin)
- **`core/`** - Shared utilities (DB, config, auth)

---

## Endpoint Design Patterns

### 1. Naming Conventions

**Use RESTful patterns with consistent naming:**

```python
# ✅ Good: Plural nouns, kebab-case
GET    /api/users              # List users
POST   /api/users              # Create user
GET    /api/users/{id}         # Get user by ID
PATCH  /api/users/{id}         # Update user
DELETE /api/users/{id}         # Delete user

GET    /api/user-profiles      # Kebab-case for multi-word
GET    /api/blog-posts         # Plural nouns

# ❌ Bad: Verbs, mixed case, inconsistent plurals
GET    /api/getUsers
POST   /api/createUser
GET    /api/user               # Should be plural
GET    /api/UserProfiles       # Should be lowercase
```

**Versioning (optional but recommended for public APIs):**

```python
GET /api/v1/users
GET /api/v2/users
```

### 2. Query Parameters for Filtering

```python
from fastapi import FastAPI, Query
from typing import Optional

app = FastAPI()

@app.get("/api/users")
def list_users(
    page: int = Query(1, ge=1, description="Page number"),
    page_size: int = Query(10, ge=1, le=100, description="Items per page"),
    role: Optional[str] = Query(None, description="Filter by role"),
    is_active: Optional[bool] = Query(None, description="Filter by status")
):
    """
    List users with optional filtering and pagination.

    - **page**: Page number (starts at 1)
    - **page_size**: Number of items per page (max 100)
    - **role**: Filter by user role (e.g., "admin", "user")
    - **is_active**: Filter by active status
    """
    # Implementation
    pass
```

### 3. Path Parameters for Resource Identification

```python
from fastapi import Path

@app.get("/api/users/{user_id}")
def get_user(
    user_id: int = Path(..., ge=1, description="User ID")
):
    """Get a specific user by ID."""
    # Implementation
    pass

@app.patch("/api/users/{user_id}")
def update_user(
    user_id: int = Path(..., ge=1),
    updates: UserUpdate  # Pydantic model
):
    """Update a specific user."""
    # Implementation
    pass
```

---

## Request Validation with Pydantic

### 1. Define Request Models

**File: `schemas/user.py`**

```python
from pydantic import BaseModel, Field, EmailStr, validator
from typing import Optional, List
from datetime import datetime

class UserCreate(BaseModel):
    """Payload for creating a new user."""
    email: EmailStr = Field(..., description="User email address")
    username: str = Field(..., min_length=3, max_length=50, pattern="^[a-zA-Z0-9_]+$")
    password: str = Field(..., min_length=8, description="Password (min 8 characters)")
    full_name: Optional[str] = Field(None, max_length=100)

    @validator('password')
    def password_strength(cls, v):
        """Ensure password has at least one digit and one letter."""
        if not any(char.isdigit() for char in v):
            raise ValueError('Password must contain at least one digit')
        if not any(char.isalpha() for char in v):
            raise ValueError('Password must contain at least one letter')
        return v


class UserUpdate(BaseModel):
    """Payload for updating a user (all fields optional)."""
    full_name: Optional[str] = Field(None, max_length=100)
    bio: Optional[str] = Field(None, max_length=500)
    is_active: Optional[bool] = None


class UserResponse(BaseModel):
    """Response model for user data (excludes password)."""
    id: int
    email: str
    username: str
    full_name: Optional[str]
    is_active: bool
    created_at: datetime

    class Config:
        orm_mode = True  # Allow conversion from SQLAlchemy models


class UserListResponse(BaseModel):
    """Response for list of users with pagination metadata."""
    users: List[UserResponse]
    total: int
    page: int
    page_size: int
```

### 2. Use Models in Endpoints

```python
from fastapi import HTTPException, status

@app.post("/api/users", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
def create_user(user: UserCreate):
    """
    Create a new user.

    Validates email format, username pattern, and password strength.
    Returns the created user (without password).
    """
    # Check if user exists
    if user_exists(user.email):
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="User with this email already exists"
        )

    # Create user (business logic in service layer)
    new_user = user_service.create(user)
    return new_user


@app.patch("/api/users/{user_id}", response_model=UserResponse)
def update_user(user_id: int, updates: UserUpdate):
    """
    Update an existing user.

    Only provided fields will be updated (partial update).
    """
    user = user_service.get(user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"User {user_id} not found"
        )

    # Apply updates
    updated_user = user_service.update(user_id, updates.dict(exclude_unset=True))
    return updated_user
```

**Key patterns:**
- `UserCreate` - Required fields for creation
- `UserUpdate` - Optional fields for updates (use `exclude_unset=True` for partial updates)
- `UserResponse` - What the API returns (excludes sensitive fields)
- `orm_mode = True` - Allows returning SQLAlchemy models directly

---

## Response Standardization

### 1. Standardized Response Wrapper (Optional)

For consistent API responses across all endpoints:

```python
from typing import Generic, TypeVar, Optional
from pydantic import BaseModel
from datetime import datetime

T = TypeVar('T')

class APIResponse(BaseModel, Generic[T]):
    """Standard API response wrapper."""
    success: bool
    data: Optional[T] = None
    error: Optional[str] = None
    metadata: Optional[dict] = None

    class Config:
        schema_extra = {
            "example": {
                "success": True,
                "data": {"id": 1, "name": "John"},
                "error": None,
                "metadata": {
                    "timestamp": "2026-02-06T10:30:00Z",
                    "version": "1.0"
                }
            }
        }


@app.get("/api/users/{user_id}")
def get_user(user_id: int) -> APIResponse[UserResponse]:
    """Get user with standardized response."""
    user = user_service.get(user_id)
    if not user:
        return APIResponse(
            success=False,
            error=f"User {user_id} not found"
        )

    return APIResponse(
        success=True,
        data=user,
        metadata={
            "timestamp": datetime.utcnow().isoformat(),
            "version": "1.0"
        }
    )
```

**Note:** This pattern is optional. Many APIs prefer direct responses without wrappers for simpler client code.

### 2. Direct Response (Recommended)

FastAPI's native approach (cleaner, simpler):

```python
@app.get("/api/users/{user_id}", response_model=UserResponse)
def get_user(user_id: int):
    """Get user by ID."""
    user = user_service.get(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

---

## Error Handling

### 1. HTTP Exception Patterns

```python
from fastapi import HTTPException, status

# 400: Bad Request (invalid input)
raise HTTPException(
    status_code=status.HTTP_400_BAD_REQUEST,
    detail="Invalid email format"
)

# 401: Unauthorized
raise HTTPException(
    status_code=status.HTTP_401_UNAUTHORIZED,
    detail="Invalid credentials",
    headers={"WWW-Authenticate": "Bearer"}
)

# 403: Forbidden (authenticated but not authorized)
raise HTTPException(
    status_code=status.HTTP_403_FORBIDDEN,
    detail="You don't have permission to access this resource"
)

# 404: Not Found
raise HTTPException(
    status_code=status.HTTP_404_NOT_FOUND,
    detail=f"User {user_id} not found"
)

# 409: Conflict (duplicate resource)
raise HTTPException(
    status_code=status.HTTP_409_CONFLICT,
    detail="User with this email already exists"
)

# 422: Validation Error (automatically raised by Pydantic)
# No need to manually raise - FastAPI handles this

# 500: Internal Server Error
raise HTTPException(
    status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
    detail="An unexpected error occurred"
)
```

### 2. Custom Exception Handlers

```python
from fastapi import Request
from fastapi.responses import JSONResponse

class DatabaseException(Exception):
    """Custom exception for database errors."""
    def __init__(self, message: str):
        self.message = message

@app.exception_handler(DatabaseException)
async def database_exception_handler(request: Request, exc: DatabaseException):
    """Handle database errors gracefully."""
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={
            "error": "Database error",
            "detail": exc.message
        }
    )

@app.exception_handler(Exception)
async def general_exception_handler(request: Request, exc: Exception):
    """Catch-all for unexpected errors."""
    # Log the error
    logger.exception("Unexpected error occurred")

    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={
            "error": "Internal server error",
            "detail": "An unexpected error occurred. Please try again later."
        }
    )
```

### 3. Try-Except in Endpoints

```python
@app.post("/api/users", response_model=UserResponse)
def create_user(user: UserCreate):
    """Create a new user with error handling."""
    try:
        # Check if user exists
        if user_service.exists(user.email):
            raise HTTPException(
                status_code=status.HTTP_409_CONFLICT,
                detail="User with this email already exists"
            )

        # Create user
        new_user = user_service.create(user)
        return new_user

    except DatabaseException as e:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail=f"Database error: {e.message}"
        )
    except Exception as e:
        # Log unexpected errors
        logger.exception("Unexpected error in create_user")
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="An unexpected error occurred"
        )
```

---

## Database Integration

### 1. SQLAlchemy Setup with Context Manager

**File: `core/database.py`**

```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
from contextlib import contextmanager
import os

DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://user:password@localhost/dbname")

engine = create_engine(
    DATABASE_URL,
    pool_pre_ping=True,  # Check connection health
    pool_size=5,
    max_overflow=10
)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()

@contextmanager
def get_db():
    """Context manager for database sessions."""
    db = SessionLocal()
    try:
        yield db
        db.commit()
    except Exception:
        db.rollback()
        raise
    finally:
        db.close()


def init_db():
    """Initialize database tables."""
    Base.metadata.create_all(bind=engine)


def test_connection():
    """Test database connection."""
    try:
        with get_db() as db:
            db.execute("SELECT 1")
        return True
    except Exception as e:
        print(f"Database connection failed: {e}")
        return False
```

### 2. Using Database in Endpoints

```python
from core.database import get_db
from models.user import User

@app.get("/api/users", response_model=UserListResponse)
def list_users(
    page: int = Query(1, ge=1),
    page_size: int = Query(10, ge=1, le=100)
):
    """List users with pagination."""
    with get_db() as db:
        # Calculate offset
        offset = (page - 1) * page_size

        # Query with pagination
        query = db.query(User).filter(User.is_active == True)
        total = query.count()
        users = query.offset(offset).limit(page_size).all()

        return UserListResponse(
            users=users,
            total=total,
            page=page,
            page_size=page_size
        )


@app.patch("/api/users/{user_id}", response_model=UserResponse)
def update_user(user_id: int, updates: UserUpdate):
    """Update user with database transaction."""
    with get_db() as db:
        # Find user
        user = db.query(User).filter(User.id == user_id).first()

        if not user:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail=f"User {user_id} not found"
            )

        # Apply updates
        update_data = updates.dict(exclude_unset=True)
        for key, value in update_data.items():
            setattr(user, key, value)

        db.commit()
        db.refresh(user)

        return user
```

### 3. Startup Event for Database Initialization

```python
@app.on_event("startup")
async def startup_event():
    """Initialize database on application startup."""
    print("Testing database connection...")
    if test_connection():
        init_db()
        print("Database initialized successfully")
    else:
        print("Warning: Database connection failed")
```

---

## CORS Configuration

### Development Setup (Allow All)

```python
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Allow all origins (dev only!)
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Production Setup (Whitelist Origins)

```python
import os

ALLOWED_ORIGINS = os.getenv("ALLOWED_ORIGINS", "https://myapp.com,https://www.myapp.com").split(",")

app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS,  # Whitelist specific domains
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Content-Type", "Authorization"],
)
```

### Environment-Based CORS

```python
from fastapi.middleware.cors import CORSMiddleware
import os

def get_cors_origins():
    """Get CORS origins based on environment."""
    env = os.getenv("ENV", "development")

    if env == "development":
        return ["http://localhost:3000", "http://localhost:5173"]
    elif env == "staging":
        return ["https://staging.myapp.com"]
    elif env == "production":
        return ["https://myapp.com", "https://www.myapp.com"]
    else:
        return []

app.add_middleware(
    CORSMiddleware,
    allow_origins=get_cors_origins(),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

---

## Async Patterns

### 1. When to Use Async

**Use async for I/O-bound operations:**
- Database queries
- External API calls
- File I/O
- Network requests

**Don't use async for:**
- CPU-bound tasks (use `run_in_executor` instead)
- Synchronous database drivers (use sync endpoints)

### 2. Async Endpoints

```python
import httpx
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

@app.get("/api/users/{user_id}/posts")
async def get_user_posts(user_id: int):
    """Async endpoint for fetching user posts."""
    async with httpx.AsyncClient() as client:
        response = await client.get(f"https://api.example.com/users/{user_id}/posts")
        return response.json()


@app.post("/api/users")
async def create_user_async(user: UserCreate):
    """Async database operation."""
    async with async_session_factory() as session:
        new_user = User(**user.dict())
        session.add(new_user)
        await session.commit()
        await session.refresh(new_user)
        return new_user
```

### 3. Background Tasks

For long-running operations:

```python
from fastapi import BackgroundTasks

def send_welcome_email(email: str):
    """Send welcome email (runs in background)."""
    # Email sending logic
    print(f"Sending welcome email to {email}")

@app.post("/api/users", status_code=status.HTTP_201_CREATED)
async def create_user(user: UserCreate, background_tasks: BackgroundTasks):
    """Create user and send welcome email in background."""
    # Create user
    new_user = user_service.create(user)

    # Schedule background task
    background_tasks.add_task(send_welcome_email, new_user.email)

    return new_user
```

---

## API Documentation

### 1. Endpoint Docstrings

FastAPI automatically generates OpenAPI docs from docstrings:

```python
@app.post("/api/users", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
def create_user(user: UserCreate):
    """
    Create a new user account.

    This endpoint creates a new user with the provided information.
    The password will be hashed before storing in the database.

    - **email**: Valid email address (required)
    - **username**: Alphanumeric username, 3-50 characters (required)
    - **password**: Minimum 8 characters with at least one digit and one letter (required)
    - **full_name**: User's full name (optional)

    Returns the created user object (without password).

    Raises:
        400: Invalid input or email already exists
        500: Internal server error
    """
    # Implementation
    pass
```

### 2. Response Examples

Add examples to Pydantic models:

```python
class UserResponse(BaseModel):
    id: int
    email: str
    username: str
    full_name: Optional[str]
    is_active: bool
    created_at: datetime

    class Config:
        orm_mode = True
        schema_extra = {
            "example": {
                "id": 1,
                "email": "john@example.com",
                "username": "johndoe",
                "full_name": "John Doe",
                "is_active": True,
                "created_at": "2026-02-06T10:30:00Z"
            }
        }
```

### 3. Tags and Metadata

Organize endpoints with tags:

```python
app = FastAPI(
    title="My API",
    description="A production-ready FastAPI application",
    version="1.0.0",
    contact={
        "name": "API Support",
        "email": "support@myapi.com"
    },
    license_info={
        "name": "MIT"
    }
)

@app.post("/api/users", tags=["users"], summary="Create a new user")
def create_user(user: UserCreate):
    """Create a new user account."""
    pass

@app.get("/api/posts", tags=["posts"], summary="List all posts")
def list_posts():
    """Get a list of all blog posts."""
    pass
```

Access docs at:
- **Swagger UI**: `http://localhost:8000/docs`
- **ReDoc**: `http://localhost:8000/redoc`
- **OpenAPI JSON**: `http://localhost:8000/openapi.json`

---

## Performance Optimization

### 1. Caching with Redis

```python
import redis
import json
from functools import wraps

redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def cache(expire_seconds=300):
    """Cache decorator for expensive operations."""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Generate cache key
            cache_key = f"{func.__name__}:{str(args)}:{str(kwargs)}"

            # Check cache
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)

            # Call function
            result = await func(*args, **kwargs)

            # Store in cache
            redis_client.setex(cache_key, expire_seconds, json.dumps(result))

            return result
        return wrapper
    return decorator

@app.get("/api/users")
@cache(expire_seconds=60)
async def list_users():
    """List users with caching."""
    # Expensive database query
    return get_all_users()
```

### 2. Database Query Optimization

```python
from sqlalchemy.orm import joinedload

@app.get("/api/users/{user_id}/posts")
def get_user_with_posts(user_id: int):
    """Eager load related data to avoid N+1 queries."""
    with get_db() as db:
        # ✅ Good: Single query with join
        user = db.query(User)\
            .options(joinedload(User.posts))\
            .filter(User.id == user_id)\
            .first()

        return user
```

### 3. Rate Limiting

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.get("/api/users")
@limiter.limit("100/minute")
def list_users(request: Request):
    """List users with rate limiting (100 requests per minute)."""
    return get_all_users()
```

### 4. Pagination

Always paginate list endpoints:

```python
@app.get("/api/users", response_model=UserListResponse)
def list_users(
    page: int = Query(1, ge=1, description="Page number"),
    page_size: int = Query(10, ge=1, le=100, description="Items per page")
):
    """List users with pagination."""
    with get_db() as db:
        offset = (page - 1) * page_size

        total = db.query(User).count()
        users = db.query(User)\
            .offset(offset)\
            .limit(page_size)\
            .all()

        return UserListResponse(
            users=users,
            total=total,
            page=page,
            page_size=page_size
        )
```

---

## Complete Example

Putting it all together:

**File: `main.py`**

```python
from fastapi import FastAPI, HTTPException, Query, Path, status, BackgroundTasks
from fastapi.middleware.cors import CORSMiddleware
from typing import Optional
import os

from core.database import get_db, init_db, test_connection
from models.user import User
from schemas.user import UserCreate, UserUpdate, UserResponse, UserListResponse

app = FastAPI(
    title="User API",
    description="A production-ready user management API",
    version="1.0.0"
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Startup
@app.on_event("startup")
async def startup_event():
    """Initialize database on startup."""
    if test_connection():
        init_db()
        print("Database initialized")

# Health check
@app.get("/health")
def health_check():
    """Health check endpoint."""
    return {"status": "healthy"}

# List users
@app.get("/api/users", response_model=UserListResponse, tags=["users"])
def list_users(
    page: int = Query(1, ge=1),
    page_size: int = Query(10, ge=1, le=100),
    is_active: Optional[bool] = None
):
    """
    List users with optional filtering and pagination.

    - **page**: Page number (starts at 1)
    - **page_size**: Items per page (max 100)
    - **is_active**: Filter by active status
    """
    with get_db() as db:
        query = db.query(User)

        if is_active is not None:
            query = query.filter(User.is_active == is_active)

        total = query.count()
        offset = (page - 1) * page_size
        users = query.offset(offset).limit(page_size).all()

        return UserListResponse(
            users=users,
            total=total,
            page=page,
            page_size=page_size
        )

# Get user
@app.get("/api/users/{user_id}", response_model=UserResponse, tags=["users"])
def get_user(user_id: int = Path(..., ge=1)):
    """Get a specific user by ID."""
    with get_db() as db:
        user = db.query(User).filter(User.id == user_id).first()

        if not user:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail=f"User {user_id} not found"
            )

        return user

# Create user
@app.post("/api/users", response_model=UserResponse, status_code=status.HTTP_201_CREATED, tags=["users"])
def create_user(user: UserCreate, background_tasks: BackgroundTasks):
    """
    Create a new user account.

    Validates email format and password strength.
    Sends a welcome email in the background.
    """
    with get_db() as db:
        # Check if user exists
        existing = db.query(User).filter(User.email == user.email).first()
        if existing:
            raise HTTPException(
                status_code=status.HTTP_409_CONFLICT,
                detail="User with this email already exists"
            )

        # Create user
        new_user = User(**user.dict())
        db.add(new_user)
        db.commit()
        db.refresh(new_user)

        # Send welcome email in background
        background_tasks.add_task(send_welcome_email, new_user.email)

        return new_user

# Update user
@app.patch("/api/users/{user_id}", response_model=UserResponse, tags=["users"])
def update_user(
    user_id: int = Path(..., ge=1),
    updates: UserUpdate = None
):
    """
    Update an existing user.

    Only provided fields will be updated (partial update).
    """
    with get_db() as db:
        user = db.query(User).filter(User.id == user_id).first()

        if not user:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail=f"User {user_id} not found"
            )

        # Apply updates
        update_data = updates.dict(exclude_unset=True)
        for key, value in update_data.items():
            setattr(user, key, value)

        db.commit()
        db.refresh(user)

        return user

# Delete user
@app.delete("/api/users/{user_id}", status_code=status.HTTP_204_NO_CONTENT, tags=["users"])
def delete_user(user_id: int = Path(..., ge=1)):
    """Delete a user by ID."""
    with get_db() as db:
        user = db.query(User).filter(User.id == user_id).first()

        if not user:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail=f"User {user_id} not found"
            )

        db.delete(user)
        db.commit()

# Background task
def send_welcome_email(email: str):
    """Send welcome email (background task)."""
    print(f"Sending welcome email to {email}")
    # Email sending logic here
```

---

## Summary

**Key Takeaways:**

1. **Structure** - Separate concerns (models, schemas, services, endpoints)
2. **Validation** - Use Pydantic models with proper validators
3. **Error Handling** - Use appropriate HTTP status codes and custom exception handlers
4. **Database** - Use context managers for safe transaction handling
5. **CORS** - Whitelist origins in production
6. **Async** - Use for I/O-bound operations
7. **Documentation** - Add docstrings and examples for auto-generated docs
8. **Performance** - Cache expensive operations, paginate lists, eager load relations

## Further Reading

- [FastAPI Official Documentation](https://fastapi.tiangolo.com/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [SQLAlchemy ORM Tutorial](https://docs.sqlalchemy.org/en/20/orm/tutorial.html)
- [HTTP Status Codes Reference](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)

---

**Created:** 2026-02-06
**Tags:** #fastapi #python #backend #api #rest #pydantic #sqlalchemy
