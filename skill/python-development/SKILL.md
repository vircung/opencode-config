---
name: python-development
description: Modern Python coding standards, best practices, testing patterns, and implementation guidelines
license: MIT
compatibility: opencode
metadata:
  audience: developers
  language: python
  version: "3.10+"
---

# Python Development Skill

## Modern Python Standards (3.10+)

### Type Hints and Annotations
```python
from typing import Protocol, TypeVar, Generic
from collections.abc import Sequence, Mapping
from pathlib import Path

# Modern union syntax (3.10+)
def process_data(data: str | int | None) -> str:
    match data:
        case str() if data.strip():
            return f"String: {data}"
        case int() if data > 0:
            return f"Positive int: {data}"
        case int() if data <= 0:
            return f"Non-positive int: {data}"
        case None:
            return "No data"
        case _:
            return "Invalid data"

# Protocol for structural typing
class Drawable(Protocol):
    def draw(self) -> None: ...
    def area(self) -> float: ...

def render_shape(shape: Drawable) -> None:
    print(f"Rendering shape with area: {shape.area()}")
    shape.draw()

# Generic types
T = TypeVar('T')

class Repository(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []
    
    def add(self, item: T) -> None:
        self._items.append(item)
    
    def get_all(self) -> list[T]:
        return self._items.copy()
```

### Modern Python Idioms

**Dataclasses and Pydantic**
```python
from dataclasses import dataclass, field
from typing import Optional
from datetime import datetime

@dataclass
class User:
    name: str
    email: str
    created_at: datetime = field(default_factory=datetime.now)
    active: bool = True
    tags: list[str] = field(default_factory=list)
    
    def __post_init__(self):
        if "@" not in self.email:
            raise ValueError("Invalid email format")

# Pydantic for validation (external data)
from pydantic import BaseModel, EmailStr, Field

class UserModel(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    email: EmailStr
    age: int = Field(ge=0, le=150)
    
    class Config:
        validate_assignment = True
```

**Context Managers**
```python
from contextlib import contextmanager
import sqlite3

@contextmanager
def database_transaction(db_path: str):
    conn = sqlite3.connect(db_path)
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()

# Usage
with database_transaction("app.db") as conn:
    conn.execute("INSERT INTO users (name) VALUES (?)", ("Alice",))
```

**Pathlib Usage**
```python
from pathlib import Path

def process_config_files(config_dir: str | Path) -> dict[str, str]:
    config_path = Path(config_dir)
    
    if not config_path.exists():
        raise FileNotFoundError(f"Config directory not found: {config_path}")
    
    configs = {}
    for config_file in config_path.glob("*.yaml"):
        with config_file.open() as f:
            configs[config_file.stem] = f.read()
    
    return configs
```

## Error Handling Patterns

### Custom Exceptions
```python
class AppException(Exception):
    """Base exception for application-specific errors."""
    pass

class ValidationError(AppException):
    """Raised when data validation fails."""
    def __init__(self, field: str, value: any, message: str):
        self.field = field
        self.value = value
        super().__init__(f"Validation failed for {field}: {message}")

class DatabaseError(AppException):
    """Raised when database operations fail."""
    pass

def validate_user_age(age: int) -> None:
    if age < 0 or age > 150:
        raise ValidationError("age", age, "must be between 0 and 150")
```

### Error Handling Best Practices
```python
import logging
from typing import Optional

logger = logging.getLogger(__name__)

def safe_divide(a: float, b: float) -> Optional[float]:
    """Safely divide two numbers, returning None if division by zero."""
    try:
        return a / b
    except ZeroDivisionError:
        logger.warning(f"Division by zero attempted: {a} / {b}")
        return None

def process_user_data(data: dict) -> dict:
    """Process user data with comprehensive error handling."""
    try:
        # Validate required fields
        if not (name := data.get("name", "").strip()):
            raise ValidationError("name", data.get("name"), "is required")
        
        if not (email := data.get("email", "").strip()):
            raise ValidationError("email", data.get("email"), "is required")
        
        # Process data
        return {
            "name": name.title(),
            "email": email.lower(),
            "created_at": datetime.now().isoformat()
        }
        
    except ValidationError:
        logger.error(f"User data validation failed: {data}")
        raise
    except Exception as e:
        logger.exception("Unexpected error processing user data")
        raise AppException(f"Failed to process user data: {e}") from e
```

## Testing Patterns

### Pytest Best Practices
```python
import pytest
from unittest.mock import Mock, patch
from pathlib import Path

# Fixtures
@pytest.fixture
def user_data():
    return {
        "name": "Alice Smith",
        "email": "alice@example.com",
        "age": 30
    }

@pytest.fixture
def temp_config_dir(tmp_path):
    """Create a temporary config directory with test files."""
    config_dir = tmp_path / "config"
    config_dir.mkdir()
    
    (config_dir / "app.yaml").write_text("debug: true\nport: 8000\n")
    (config_dir / "db.yaml").write_text("host: localhost\nport: 5432\n")
    
    return config_dir

# Parametrized tests
@pytest.mark.parametrize("input_age,expected", [
    (25, True),
    (0, True),
    (150, True),
    (-1, False),
    (151, False),
])
def test_age_validation(input_age, expected):
    if expected:
        validate_user_age(input_age)  # Should not raise
    else:
        with pytest.raises(ValidationError):
            validate_user_age(input_age)

# Mocking
def test_database_operation():
    with patch('app.database.connect') as mock_connect:
        mock_conn = Mock()
        mock_connect.return_value = mock_conn
        
        result = some_database_operation()
        
        mock_connect.assert_called_once()
        mock_conn.execute.assert_called_with("SELECT * FROM users")

# Async testing
@pytest.mark.asyncio
async def test_async_api_call():
    with patch('httpx.AsyncClient.get') as mock_get:
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {"success": True}
        mock_get.return_value = mock_response
        
        result = await fetch_user_data("123")
        
        assert result["success"] is True
        mock_get.assert_called_once()
```

### Test Organization
```python
# tests/conftest.py - Shared fixtures
import pytest
from app import create_app

@pytest.fixture
def app():
    return create_app(testing=True)

@pytest.fixture
def client(app):
    return app.test_client()

# tests/test_models.py - Model tests
class TestUser:
    def test_user_creation(self, user_data):
        user = User(**user_data)
        assert user.name == "Alice Smith"
        assert user.email == "alice@example.com"
    
    def test_invalid_email_raises_error(self):
        with pytest.raises(ValidationError):
            User(name="Test", email="invalid-email")

# tests/test_services.py - Service tests
class TestUserService:
    @pytest.fixture(autouse=True)
    def setup(self):
        self.service = UserService()
    
    def test_create_user_success(self, user_data):
        user = self.service.create_user(user_data)
        assert user.id is not None
        assert user.name == user_data["name"]
```

## Performance and Optimization

### Profiling and Timing
```python
import time
import functools
from typing import Callable, Any

def timer(func: Callable) -> Callable:
    """Decorator to time function execution."""
    @functools.wraps(func)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        start = time.perf_counter()
        result = func(*args, **kwargs)
        end = time.perf_counter()
        print(f"{func.__name__} took {end - start:.4f} seconds")
        return result
    return wrapper

@timer
def slow_operation(n: int) -> list[int]:
    return [i**2 for i in range(n)]

# Memory profiling with tracemalloc
import tracemalloc

def profile_memory(func: Callable) -> Callable:
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        tracemalloc.start()
        result = func(*args, **kwargs)
        current, peak = tracemalloc.get_traced_memory()
        tracemalloc.stop()
        print(f"{func.__name__} - Current: {current / 1024 / 1024:.2f}MB, Peak: {peak / 1024 / 1024:.2f}MB")
        return result
    return wrapper
```

### Async Best Practices
```python
import asyncio
import aiohttp
from typing import List

async def fetch_url(session: aiohttp.ClientSession, url: str) -> dict:
    """Fetch a single URL."""
    async with session.get(url) as response:
        return await response.json()

async def fetch_multiple_urls(urls: List[str]) -> List[dict]:
    """Fetch multiple URLs concurrently."""
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_url(session, url) for url in urls]
        return await asyncio.gather(*tasks, return_exceptions=True)

# Semaphore for rate limiting
async def fetch_with_limit(urls: List[str], max_concurrent: int = 10) -> List[dict]:
    """Fetch URLs with concurrency limit."""
    semaphore = asyncio.Semaphore(max_concurrent)
    
    async def fetch_limited(url: str) -> dict:
        async with semaphore:
            async with aiohttp.ClientSession() as session:
                return await fetch_url(session, url)
    
    tasks = [fetch_limited(url) for url in urls]
    return await asyncio.gather(*tasks, return_exceptions=True)
```

## Configuration and Environment Management

### Configuration Pattern
```python
from dataclasses import dataclass
from pathlib import Path
import os
from typing import Optional

@dataclass
class DatabaseConfig:
    host: str = "localhost"
    port: int = 5432
    username: str = "user"
    password: str = ""
    database: str = "app_db"
    
    @property
    def url(self) -> str:
        return f"postgresql://{self.username}:{self.password}@{self.host}:{self.port}/{self.database}"

@dataclass
class AppConfig:
    debug: bool = False
    secret_key: str = ""
    database: DatabaseConfig = DatabaseConfig()
    
    @classmethod
    def from_env(cls) -> "AppConfig":
        return cls(
            debug=os.getenv("DEBUG", "false").lower() == "true",
            secret_key=os.getenv("SECRET_KEY", "dev-key"),
            database=DatabaseConfig(
                host=os.getenv("DB_HOST", "localhost"),
                port=int(os.getenv("DB_PORT", "5432")),
                username=os.getenv("DB_USER", "user"),
                password=os.getenv("DB_PASSWORD", ""),
                database=os.getenv("DB_NAME", "app_db"),
            )
        )

# Usage
config = AppConfig.from_env()
```

### Logging Configuration
```python
import logging.config
import sys

LOGGING_CONFIG = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "standard": {
            "format": "%(asctime)s [%(levelname)s] %(name)s: %(message)s"
        },
        "detailed": {
            "format": "%(asctime)s [%(levelname)s] %(name)s:%(lineno)d - %(message)s"
        }
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "level": "INFO",
            "formatter": "standard",
            "stream": sys.stdout
        },
        "file": {
            "class": "logging.FileHandler",
            "level": "DEBUG",
            "formatter": "detailed",
            "filename": "app.log"
        }
    },
    "loggers": {
        "": {  # Root logger
            "level": "INFO",
            "handlers": ["console", "file"]
        },
        "app": {
            "level": "DEBUG",
            "handlers": ["console", "file"],
            "propagate": False
        }
    }
}

def setup_logging():
    logging.config.dictConfig(LOGGING_CONFIG)

# Usage in modules
logger = logging.getLogger(__name__)
logger.info("Application started")
```

These patterns provide a solid foundation for writing maintainable, tested, and performant Python code following modern best practices.