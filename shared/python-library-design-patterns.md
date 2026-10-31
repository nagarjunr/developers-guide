# Python Library Design Patterns: Building Clean, Testable Code

A comprehensive guide to designing Python libraries with explicit configuration, dependency injection, and clean architecture patterns.

## Table of Contents

1. [Design Philosophy](#design-philosophy)
2. [Explicit vs Implicit Configuration](#explicit-vs-implicit-configuration)
3. [Singleton Pattern for Resources](#singleton-pattern-for-resources)
4. [Context Managers for Resource Cleanup](#context-managers-for-resource-cleanup)
5. [Dependency Injection](#dependency-injection)
6. [Separation of Concerns](#separation-of-concerns)
7. [Complete Example](#complete-example)
8. [Testing Patterns](#testing-patterns)

---

## Design Philosophy

### Core Principles

**Good library design follows these principles:**

1. **Explicit over Implicit**
   - Configuration passed as parameters, not hidden in environment variables
   - No global state or hidden dependencies
   - Clear interfaces and contracts

2. **Deterministic Behavior**
   - Same inputs always produce same outputs
   - No side effects in constructors
   - Predictable resource lifecycle

3. **Easy to Test**
   - Pure functions where possible
   - Dependency injection for external resources
   - Mockable interfaces

4. **Production Ready**
   - Safe for batch jobs, CI/CD, containers
   - Thread-safe when needed
   - Proper resource cleanup

**Anti-patterns to avoid:**
```python
# ❌ Bad: Hidden environment variable access
class DatabaseClient:
    def __init__(self):
        self.host = os.getenv("DB_HOST")  # Implicit!
        self.port = int(os.getenv("DB_PORT", "5432"))

# ✅ Good: Explicit configuration
class DatabaseClient:
    def __init__(self, host: str, port: int = 5432):
        self.host = host
        self.port = port
```

---

## Explicit vs Implicit Configuration

### The Problem with Implicit Configuration

```python
# ❌ Bad: Library loads .env internally
# File: mylib/client.py
from dotenv import load_dotenv
import os

class APIClient:
    def __init__(self):
        load_dotenv()  # Hidden side effect!
        self.api_key = os.getenv("API_KEY")
        self.endpoint = os.getenv("API_ENDPOINT")

    def call_api(self):
        # Uses self.api_key implicitly
        pass
```

**Problems:**
- ❌ Can't use multiple configurations in one process
- ❌ Hard to test (relies on environment state)
- ❌ No IDE autocomplete for parameters
- ❌ Unexpected behavior in containers/CI
- ❌ Doesn't work in Airflow/batch jobs

### The Solution: Explicit Configuration

```python
# ✅ Good: Explicit parameter injection
# File: mylib/client.py
class APIClient:
    def __init__(self, api_key: str, endpoint: str):
        """
        Initialize API client.

        Args:
            api_key: API authentication key
            endpoint: Base API endpoint URL
        """
        self.api_key = api_key
        self.endpoint = endpoint

    def call_api(self, path: str) -> dict:
        # Uses explicit parameters
        url = f"{self.endpoint}/{path}"
        headers = {"Authorization": f"Bearer {self.api_key}"}
        # Make request...
        return {}
```

**Application layer handles configuration:**

```python
# File: app/main.py
from dotenv import load_dotenv
import os
from mylib import APIClient

# Load environment at application startup
load_dotenv()

# Create client with explicit configuration
client = APIClient(
    api_key=os.getenv("API_KEY"),
    endpoint=os.getenv("API_ENDPOINT")
)

# Use client
result = client.call_api("/users")
```

**Benefits:**
- ✅ Clear dependencies (visible in constructor)
- ✅ Easy to test (pass any values)
- ✅ Can create multiple instances with different configs
- ✅ Works in any environment (Docker, Airflow, CI/CD)
- ✅ IDE autocomplete and type checking

---

## Singleton Pattern for Resources

### When to Use Singletons

Use singletons for:
- Database connection pools
- HTTP session objects
- Expensive one-time initializations
- Shared cache instances

**Don't use singletons for:**
- Business logic
- Stateless services
- Per-request data

### Singleton Implementation

```python
# File: app/connectors.py
from typing import Optional
import os
from dotenv import load_dotenv

from mylib import DatabaseClient, CacheClient, APIClient

# Load environment once at module level
load_dotenv()

# Singleton instances
_db_client: Optional[DatabaseClient] = None
_cache_client: Optional[CacheClient] = None
_api_client: Optional[APIClient] = None


def get_db_client() -> DatabaseClient:
    """
    Get or create database client singleton.

    Thread-safe initialization on first call.
    """
    global _db_client
    if _db_client is None:
        _db_client = DatabaseClient(
            host=os.getenv("DB_HOST", "localhost"),
            port=int(os.getenv("DB_PORT", "5432")),
            database=os.getenv("DB_NAME", "mydb"),
            user=os.getenv("DB_USER"),
            password=os.getenv("DB_PASSWORD"),
        )
    return _db_client


def get_cache_client() -> CacheClient:
    """Get or create cache client singleton."""
    global _cache_client
    if _cache_client is None:
        _cache_client = CacheClient(
            host=os.getenv("REDIS_HOST", "localhost"),
            port=int(os.getenv("REDIS_PORT", "6379")),
        )
    return _cache_client


def get_api_client() -> APIClient:
    """Get or create API client singleton."""
    global _api_client
    if _api_client is None:
        _api_client = APIClient(
            api_key=os.getenv("API_KEY"),
            endpoint=os.getenv("API_ENDPOINT"),
        )
    return _api_client


def close_all_connections():
    """Close all connections on shutdown."""
    global _db_client, _cache_client, _api_client

    for client in [_db_client, _cache_client, _api_client]:
        if client and hasattr(client, 'close'):
            client.close()

    # Reset singletons
    _db_client = None
    _cache_client = None
    _api_client = None
```

### Using Singletons in Application

```python
# File: app/main.py
from fastapi import FastAPI
from app.connectors import get_db_client, get_api_client, close_all_connections

app = FastAPI()

@app.on_event("shutdown")
async def shutdown_event():
    """Clean up resources on application shutdown."""
    close_all_connections()


@app.get("/users")
def list_users():
    """List users from database."""
    db = get_db_client()  # Returns same instance every time
    users = db.query("SELECT * FROM users")
    return users


@app.get("/external-data")
def get_external_data():
    """Fetch data from external API."""
    api = get_api_client()  # Returns same instance every time
    data = api.call_api("/data")
    return data
```

### Thread-Safe Singleton (Advanced)

For multi-threaded applications:

```python
import threading
from typing import Optional

_db_client: Optional[DatabaseClient] = None
_lock = threading.Lock()


def get_db_client() -> DatabaseClient:
    """Thread-safe singleton."""
    global _db_client

    if _db_client is None:
        with _lock:
            # Double-check locking pattern
            if _db_client is None:
                _db_client = DatabaseClient(
                    host=os.getenv("DB_HOST"),
                    port=int(os.getenv("DB_PORT", "5432")),
                )

    return _db_client
```

---

## Context Managers for Resource Cleanup

### Why Context Managers?

Context managers ensure resources are properly cleaned up, even if exceptions occur.

```python
# ❌ Bad: Manual cleanup (can leak resources)
def process_file():
    file = open("data.txt")
    data = file.read()
    file.close()  # Won't run if error occurs above!
    return data

# ✅ Good: Automatic cleanup with context manager
def process_file():
    with open("data.txt") as file:
        data = file.read()
    return data  # File always closed, even on error
```

### Implementing Context Managers

**Pattern 1: Class-based context manager**

```python
from typing import Optional

class DatabaseConnection:
    """Database connection with automatic cleanup."""

    def __init__(self, host: str, port: int, database: str):
        self.host = host
        self.port = port
        self.database = database
        self._connection: Optional[object] = None

    def __enter__(self):
        """Called when entering 'with' block."""
        print(f"Connecting to {self.host}:{self.port}/{self.database}")
        # Establish connection
        self._connection = self._connect()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Called when exiting 'with' block (even on error)."""
        if self._connection:
            print("Closing connection")
            self._connection.close()
        return False  # Don't suppress exceptions

    def _connect(self):
        """Establish database connection."""
        # Connection logic here
        return object()  # Placeholder

    def query(self, sql: str):
        """Execute query."""
        if not self._connection:
            raise RuntimeError("Not connected")
        # Query logic here
        return []


# Usage
with DatabaseConnection("localhost", 5432, "mydb") as db:
    results = db.query("SELECT * FROM users")
    # Connection automatically closed after this block
```

**Pattern 2: Generator-based context manager**

```python
from contextlib import contextmanager

@contextmanager
def database_session(host: str, port: int):
    """Context manager for database sessions."""
    print("Opening session")
    connection = create_connection(host, port)

    try:
        yield connection  # Provide connection to 'with' block
    finally:
        print("Closing session")
        connection.close()


# Usage
with database_session("localhost", 5432) as conn:
    conn.execute("SELECT * FROM users")
    # Connection automatically closed
```

### Real-World Example: Database Session Manager

```python
from contextlib import contextmanager
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session

class DatabaseClient:
    """Database client with session management."""

    def __init__(self, connection_string: str):
        self.engine = create_engine(connection_string)
        self.SessionLocal = sessionmaker(bind=self.engine)

    @contextmanager
    def get_session(self) -> Session:
        """
        Context manager for database sessions.

        Automatically commits on success, rolls back on error.
        """
        session = self.SessionLocal()
        try:
            yield session
            session.commit()
        except Exception:
            session.rollback()
            raise
        finally:
            session.close()

    def query(self, sql: str):
        """Execute query with automatic session management."""
        with self.get_session() as session:
            result = session.execute(sql)
            return result.fetchall()


# Usage
db = DatabaseClient("postgresql://user:pass@localhost/mydb")

# Pattern 1: Use context manager directly
with db.get_session() as session:
    users = session.query(User).all()
    # Session auto-commits on success, rolls back on error

# Pattern 2: Use convenience method
results = db.query("SELECT * FROM users")
```

---

## Dependency Injection

### What is Dependency Injection?

**Instead of creating dependencies internally, accept them as parameters.**

```python
# ❌ Bad: Hard-coded dependencies
class UserService:
    def __init__(self):
        self.db = DatabaseClient("localhost", 5432)  # Hard-coded!
        self.cache = CacheClient("localhost", 6379)  # Hard-coded!

    def get_user(self, user_id: int):
        # Uses hard-coded dependencies
        return self.db.query(f"SELECT * FROM users WHERE id = {user_id}")


# ✅ Good: Dependency injection
class UserService:
    def __init__(self, db: DatabaseClient, cache: CacheClient):
        self.db = db
        self.cache = cache

    def get_user(self, user_id: int):
        # Uses injected dependencies
        return self.db.query(f"SELECT * FROM users WHERE id = {user_id}")


# Create dependencies
db = DatabaseClient("localhost", 5432)
cache = CacheClient("localhost", 6379)

# Inject dependencies
service = UserService(db=db, cache=cache)
```

**Benefits:**
- ✅ Easy to test (inject mocks)
- ✅ Flexible (can swap implementations)
- ✅ Clear dependencies (visible in constructor)
- ✅ No hidden coupling

### Dependency Injection Patterns

**Pattern 1: Constructor injection (recommended)**

```python
class ReportGenerator:
    def __init__(self, db: DatabaseClient, storage: StorageClient):
        self.db = db
        self.storage = storage

    def generate_report(self):
        data = self.db.query("SELECT * FROM sales")
        report = self._create_report(data)
        self.storage.upload("report.pdf", report)
        return report


# Usage
report_gen = ReportGenerator(
    db=get_db_client(),
    storage=get_storage_client()
)
report_gen.generate_report()
```

**Pattern 2: Method injection (for optional dependencies)**

```python
class DataProcessor:
    def process(self, data: list, logger: Optional[Logger] = None):
        """Process data with optional logger injection."""
        if logger:
            logger.info("Starting processing")

        result = [x * 2 for x in data]

        if logger:
            logger.info(f"Processed {len(result)} items")

        return result


# Usage with logger
processor = DataProcessor()
result = processor.process([1, 2, 3], logger=get_logger())

# Usage without logger
result = processor.process([1, 2, 3])
```

**Pattern 3: Factory pattern**

```python
class ClientFactory:
    """Factory for creating configured clients."""

    def __init__(self, config: dict):
        self.config = config

    def create_db_client(self) -> DatabaseClient:
        return DatabaseClient(
            host=self.config["db_host"],
            port=self.config["db_port"],
        )

    def create_api_client(self) -> APIClient:
        return APIClient(
            api_key=self.config["api_key"],
            endpoint=self.config["api_endpoint"],
        )


# Usage
factory = ClientFactory(config={
    "db_host": "localhost",
    "db_port": 5432,
    "api_key": "secret",
    "api_endpoint": "https://api.example.com"
})

db = factory.create_db_client()
api = factory.create_api_client()
```

---

## Separation of Concerns

### Layer Architecture

**Good library design separates concerns into layers:**

```
Application Layer (your app)
    ↓
Service Layer (business logic)
    ↓
Client Layer (external systems)
    ↓
Base/Common Layer (utilities)
```

### Example: Multi-Layer Design

**Base layer: Common utilities**

```python
# File: mylib/_common/base_client.py
from abc import ABC, abstractmethod

class BaseClient(ABC):
    """Abstract base class for all clients."""

    @abstractmethod
    def close(self):
        """Close connections and clean up resources."""
        pass

    @abstractmethod
    def health_check(self) -> bool:
        """Check if connection is healthy."""
        pass
```

**Client layer: External system interfaces**

```python
# File: mylib/database/client.py
from mylib._common.base_client import BaseClient

class DatabaseClient(BaseClient):
    """Database client implementation."""

    def __init__(self, host: str, port: int, database: str):
        self.host = host
        self.port = port
        self.database = database
        self._connection = None

    def query(self, sql: str) -> list:
        """Execute SQL query."""
        # Query implementation
        return []

    def close(self):
        """Close database connection."""
        if self._connection:
            self._connection.close()

    def health_check(self) -> bool:
        """Check database connection."""
        try:
            self.query("SELECT 1")
            return True
        except Exception:
            return False
```

**Service layer: Business logic**

```python
# File: app/services/user_service.py
from mylib.database.client import DatabaseClient
from mylib.cache.client import CacheClient

class UserService:
    """Business logic for user operations."""

    def __init__(self, db: DatabaseClient, cache: CacheClient):
        self.db = db
        self.cache = cache

    def get_user(self, user_id: int) -> dict:
        """Get user by ID with caching."""
        # Check cache first
        cached = self.cache.get(f"user:{user_id}")
        if cached:
            return cached

        # Query database
        result = self.db.query(
            f"SELECT * FROM users WHERE id = {user_id}"
        )

        # Cache result
        if result:
            self.cache.set(f"user:{user_id}", result[0])
            return result[0]

        return None

    def create_user(self, email: str, name: str) -> dict:
        """Create new user."""
        # Validation logic
        if not email or "@" not in email:
            raise ValueError("Invalid email")

        # Insert into database
        user_id = self.db.execute(
            "INSERT INTO users (email, name) VALUES (?, ?)",
            (email, name)
        )

        # Return created user
        return {"id": user_id, "email": email, "name": name}
```

**Application layer: API endpoints**

```python
# File: app/main.py
from fastapi import FastAPI, HTTPException
from app.services.user_service import UserService
from app.connectors import get_db_client, get_cache_client

app = FastAPI()

@app.get("/users/{user_id}")
def get_user_endpoint(user_id: int):
    """API endpoint for getting user."""
    service = UserService(
        db=get_db_client(),
        cache=get_cache_client()
    )

    user = service.get_user(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    return user
```

---

## Complete Example

### Project Structure

```
my-project/
├── mylib/                    # Reusable library
│   ├── __init__.py
│   ├── _common/
│   │   ├── __init__.py
│   │   └── base_client.py
│   ├── database/
│   │   ├── __init__.py
│   │   └── client.py
│   └── cache/
│       ├── __init__.py
│       └── client.py
├── app/                      # Application code
│   ├── __init__.py
│   ├── connectors.py         # Singleton initialization
│   ├── services/
│   │   └── user_service.py
│   └── main.py               # FastAPI app
└── tests/
    ├── test_database.py
    └── test_user_service.py
```

### Implementation

**File: `mylib/_common/base_client.py`**

```python
from abc import ABC, abstractmethod

class BaseClient(ABC):
    """Base class for all clients."""

    @abstractmethod
    def close(self):
        """Close connections."""
        pass

    @abstractmethod
    def health_check(self) -> bool:
        """Check connection health."""
        pass
```

**File: `mylib/database/client.py`**

```python
from mylib._common.base_client import BaseClient

class DatabaseClient(BaseClient):
    """Database client with explicit configuration."""

    def __init__(self, host: str, port: int, database: str,
                 user: str, password: str):
        """
        Initialize database client.

        Args:
            host: Database host
            port: Database port
            database: Database name
            user: Database user
            password: Database password
        """
        self.host = host
        self.port = port
        self.database = database
        self.user = user
        self.password = password
        self._connection = None

    def query(self, sql: str) -> list:
        """Execute SELECT query."""
        # Implementation here
        return []

    def execute(self, sql: str, params: tuple = None) -> int:
        """Execute INSERT/UPDATE/DELETE query."""
        # Implementation here
        return 0

    def close(self):
        """Close database connection."""
        if self._connection:
            self._connection.close()
            self._connection = None

    def health_check(self) -> bool:
        """Check if database is accessible."""
        try:
            self.query("SELECT 1")
            return True
        except Exception:
            return False
```

**File: `app/connectors.py`**

```python
from typing import Optional
import os
from dotenv import load_dotenv

from mylib.database.client import DatabaseClient
from mylib.cache.client import CacheClient

# Load environment once
load_dotenv()

# Singletons
_db_client: Optional[DatabaseClient] = None
_cache_client: Optional[CacheClient] = None


def get_db_client() -> DatabaseClient:
    """Get database client singleton."""
    global _db_client
    if _db_client is None:
        _db_client = DatabaseClient(
            host=os.getenv("DB_HOST", "localhost"),
            port=int(os.getenv("DB_PORT", "5432")),
            database=os.getenv("DB_NAME", "mydb"),
            user=os.getenv("DB_USER"),
            password=os.getenv("DB_PASSWORD"),
        )
    return _db_client


def get_cache_client() -> CacheClient:
    """Get cache client singleton."""
    global _cache_client
    if _cache_client is None:
        _cache_client = CacheClient(
            host=os.getenv("REDIS_HOST", "localhost"),
            port=int(os.getenv("REDIS_PORT", "6379")),
        )
    return _cache_client


def close_all():
    """Close all connections."""
    global _db_client, _cache_client
    for client in [_db_client, _cache_client]:
        if client:
            client.close()
```

**File: `app/main.py`**

```python
from fastapi import FastAPI
from app.connectors import get_db_client, close_all

app = FastAPI()

@app.on_event("shutdown")
async def shutdown():
    """Clean up on shutdown."""
    close_all()

@app.get("/health")
def health_check():
    """Health check endpoint."""
    db = get_db_client()
    return {
        "status": "healthy",
        "database": db.health_check()
    }
```

---

## Testing Patterns

### Testing with Dependency Injection

```python
# test_user_service.py
from unittest.mock import Mock
import pytest

from app.services.user_service import UserService


def test_get_user_from_cache():
    """Test getting user from cache."""
    # Create mocks
    mock_db = Mock()
    mock_cache = Mock()
    mock_cache.get.return_value = {"id": 1, "name": "Alice"}

    # Inject mocks
    service = UserService(db=mock_db, cache=mock_cache)

    # Test
    user = service.get_user(1)

    # Assertions
    assert user["name"] == "Alice"
    mock_cache.get.assert_called_once_with("user:1")
    mock_db.query.assert_not_called()  # Should not hit database


def test_get_user_from_database():
    """Test getting user from database when cache misses."""
    # Create mocks
    mock_db = Mock()
    mock_db.query.return_value = [{"id": 1, "name": "Bob"}]
    mock_cache = Mock()
    mock_cache.get.return_value = None  # Cache miss

    # Inject mocks
    service = UserService(db=mock_db, cache=mock_cache)

    # Test
    user = service.get_user(1)

    # Assertions
    assert user["name"] == "Bob"
    mock_db.query.assert_called_once()
    mock_cache.set.assert_called_once()  # Should cache result
```

### Testing Context Managers

```python
def test_database_connection_cleanup():
    """Test that connection is properly closed."""
    with DatabaseConnection("localhost", 5432, "test") as db:
        results = db.query("SELECT 1")
        assert results is not None

    # Connection should be closed after exiting 'with' block
    with pytest.raises(RuntimeError, match="Not connected"):
        db.query("SELECT 1")
```

---

## Summary

**Key Design Patterns:**

1. **Explicit Configuration** - Pass parameters, don't read environment variables in libraries
2. **Singleton Pattern** - One instance for expensive resources (databases, caches)
3. **Context Managers** - Automatic resource cleanup with `with` statements
4. **Dependency Injection** - Accept dependencies as parameters for testability
5. **Separation of Concerns** - Layer architecture (client → service → application)

**Benefits:**
- ✅ Easy to test (inject mocks)
- ✅ Deterministic behavior
- ✅ Works in any environment (Docker, Airflow, CI/CD)
- ✅ Clear dependencies
- ✅ Production ready

## Further Reading

- [Python Design Patterns](https://refactoring.guru/design-patterns/python)
- [Clean Architecture in Python](https://www.cosmicpython.com/)
- [Dependency Injection in Python](https://python-dependency-injector.ets-labs.org/)
- [Context Managers in Python](https://docs.python.org/3/library/contextlib.html)

---

**Created:** 2026-02-06
**Tags:** #python #design-patterns #architecture #dependency-injection #singleton #context-managers #clean-code
