---
name: python-review
description: Python code security analysis, performance optimization, and maintainability assessment
license: MIT
compatibility: opencode
metadata:
  audience: reviewers
  language: python
  focus: security-performance
---

# Code Review Skill

## Static Analysis Tools Integration

### Pylance Integration
```python
# Pylance provides advanced type checking and code analysis
# Enable in VS Code settings or use via command line

# Type checking with Pylance
def analyze_with_pylance(file_path: str) -> None:
    """
    Run Pylance analysis via command line or check IDE integration
    """
    # Command line usage (if available):
    # pylance --check file_path
    # python -m py_analyzer file_path
    
    # Common Pylance checks:
    # - Type compatibility issues
    # - Unused imports and variables
    # - Missing type annotations
    # - Unreachable code
    # - Incorrect function signatures
    pass

# Pylance configuration for optimal analysis
# In pyrightconfig.json or pyproject.toml:
{
    "typeCheckingMode": "strict",
    "reportMissingTypeStubs": "warning",
    "reportUnusedImport": "warning",
    "reportUnusedVariable": "warning", 
    "reportDuplicateImport": "error",
    "reportUnnecessaryCast": "warning",
    "reportUnnecessaryIsInstance": "warning"
}
```

### MyPy Type Checking
```python
# mypy configuration for thorough type checking
# In mypy.ini or pyproject.toml:
[tool.mypy]
python_version = "3.10"
strict = true
warn_return_any = true
warn_unused_configs = true
warn_redundant_casts = true
warn_unused_ignores = true
warn_no_return = true
warn_unreachable = true

# Common type checking patterns
def check_type_annotations(value: str | int | None) -> str:
    # Pylance will catch type mismatches
    if isinstance(value, str):
        return value.upper()
    elif isinstance(value, int):
        return str(value)
    elif value is None:
        return "None"
    else:
        # Pylance will flag this as unreachable with proper typing
        raise TypeError("Unexpected type")
```

### Tool Integration Workflow
```bash
# Comprehensive static analysis workflow
echo "Running Python static analysis..."

# 1. Type checking with MyPy
mypy src/ --strict

# 2. Advanced analysis with Pylance (if available)
pylance --check src/ || python -m py_analyzer src/

# 3. Security analysis with Bandit
bandit -r src/ -f json

# 4. Code quality with Ruff
ruff check src/ --fix

# 5. Import sorting
ruff check --select I src/ --fix

echo "Static analysis complete"
```

## Security Analysis Checklist

### Input Validation Vulnerabilities
```python
# BAD: SQL Injection vulnerability
def get_user_by_id(user_id):
    query = f"SELECT * FROM users WHERE id = {user_id}"
    return database.execute(query)

# GOOD: Parameterized queries
def get_user_by_id(user_id: int):
    query = "SELECT * FROM users WHERE id = %s"
    return database.execute(query, (user_id,))

# BAD: Command injection
import subprocess
def process_file(filename):
    subprocess.run(f"cat {filename}", shell=True)

# GOOD: Secure command execution
import subprocess
from pathlib import Path
def process_file(filename: str):
    file_path = Path(filename).resolve()
    # Validate file path is safe
    if not file_path.is_file():
        raise ValueError("Invalid file path")
    subprocess.run(["cat", str(file_path)], check=True)
```

### Authentication and Authorization
```python
# Security checklist for auth:
# ✓ Password hashing with salt
# ✓ JWT token validation
# ✓ Session management
# ✓ Role-based access control
# ✓ Rate limiting
# ✓ HTTPS enforcement

import hashlib
import secrets
from datetime import datetime, timedelta
import jwt

class SecureAuth:
    def __init__(self, secret_key: str):
        self.secret_key = secret_key
    
    def hash_password(self, password: str) -> tuple[str, str]:
        """Return (salt, hashed_password)"""
        salt = secrets.token_hex(32)
        hashed = hashlib.pbkdf2_hmac('sha256', 
                                   password.encode(), 
                                   salt.encode(), 
                                   100000)  # 100k iterations
        return salt, hashed.hex()
    
    def verify_password(self, password: str, salt: str, stored_hash: str) -> bool:
        """Verify password against stored hash"""
        computed_hash = hashlib.pbkdf2_hmac('sha256',
                                          password.encode(),
                                          salt.encode(),
                                          100000)
        return secrets.compare_digest(computed_hash.hex(), stored_hash)
    
    def create_jwt_token(self, user_id: int, expires_hours: int = 24) -> str:
        """Create JWT token with expiration"""
        payload = {
            'user_id': user_id,
            'exp': datetime.utcnow() + timedelta(hours=expires_hours),
            'iat': datetime.utcnow()
        }
        return jwt.encode(payload, self.secret_key, algorithm='HS256')
```

### Data Exposure Prevention
```python
# BAD: Sensitive data in logs
import logging
logger = logging.getLogger(__name__)

def login_user(username, password):
    logger.info(f"Login attempt for {username} with password {password}")  # NEVER!

# GOOD: Safe logging
def login_user(username: str, password: str):
    logger.info(f"Login attempt for user: {username[:3]}***")
    # Process login without logging sensitive data

# BAD: Sensitive data in error messages
def process_payment(card_number: str, cvv: str):
    try:
        # payment processing
        pass
    except Exception as e:
        raise ValueError(f"Payment failed for card {card_number}: {e}")

# GOOD: Generic error messages
def process_payment(card_number: str, cvv: str):
    try:
        # payment processing
        pass
    except Exception as e:
        logger.error(f"Payment processing error: {e}", exc_info=True)
        raise ValueError("Payment processing failed")
```

## Performance Analysis Patterns

### Algorithmic Complexity Review
```python
# RED FLAG: O(n²) complexity
def find_duplicates_slow(items: list) -> list:
    duplicates = []
    for i, item in enumerate(items):
        for j, other in enumerate(items[i+1:], i+1):
            if item == other and item not in duplicates:
                duplicates.append(item)
    return duplicates

# OPTIMIZED: O(n) complexity
def find_duplicates_fast(items: list) -> list:
    seen = set()
    duplicates = set()
    for item in items:
        if item in seen:
            duplicates.add(item)
        else:
            seen.add(item)
    return list(duplicates)

# Memory efficiency review
def process_large_file_bad(filename: str):
    with open(filename) as f:
        lines = f.readlines()  # Loads entire file into memory
    return [line.strip() for line in lines]

def process_large_file_good(filename: str):
    with open(filename) as f:
        for line in f:  # Generator - memory efficient
            yield line.strip()
```

### Database Query Optimization
```python
# BAD: N+1 query problem
def get_users_with_posts_slow():
    users = User.objects.all()
    for user in users:
        posts = Post.objects.filter(user=user)  # N queries!
        user.posts = list(posts)
    return users

# GOOD: Use joins/prefetch
def get_users_with_posts_fast():
    return User.objects.prefetch_related('posts').all()

# Review checklist for database operations:
# ✓ Proper indexing on query fields
# ✓ Avoid N+1 queries
# ✓ Use pagination for large result sets
# ✓ Connection pooling
# ✓ Query result caching where appropriate
```

### Async/Sync Performance Patterns
```python
# BAD: Blocking I/O in async function
async def fetch_data_blocking():
    import requests
    response = requests.get("https://api.example.com")  # Blocks event loop!
    return response.json()

# GOOD: Async I/O
import aiohttp
async def fetch_data_async():
    async with aiohttp.ClientSession() as session:
        async with session.get("https://api.example.com") as response:
            return await response.json()

# BAD: Creating many async tasks without control
async def process_many_items_uncontrolled(items):
    tasks = [process_item(item) for item in items]  # Could be 10k+ tasks!
    return await asyncio.gather(*tasks)

# GOOD: Controlled concurrency
import asyncio
async def process_many_items_controlled(items, max_concurrent=50):
    semaphore = asyncio.Semaphore(max_concurrent)
    
    async def process_with_semaphore(item):
        async with semaphore:
            return await process_item(item)
    
    tasks = [process_with_semaphore(item) for item in items]
    return await asyncio.gather(*tasks)
```

## Code Quality Assessment

### Complexity Metrics
```python
# HIGH COMPLEXITY - Cyclomatic complexity > 10
def complex_function(data, option, flag, mode, debug):
    if option == 'A':
        if flag:
            if mode == 'strict':
                if debug:
                    print("Debug mode A strict")
                    # ... more nested logic
                else:
                    # ... different logic
            else:
                # ... more branches
        else:
            # ... more branches
    elif option == 'B':
        # ... more nested conditions
    # ... continues with more complexity

# BETTER: Extract functions, reduce nesting
def handle_option_a(flag: bool, mode: str, debug: bool):
    if not flag:
        return handle_option_a_no_flag()
    
    if mode == 'strict':
        return handle_strict_mode(debug)
    
    return handle_normal_mode()

def process_data(data, option: str, flag: bool, mode: str, debug: bool):
    handlers = {
        'A': lambda: handle_option_a(flag, mode, debug),
        'B': lambda: handle_option_b(flag, mode, debug),
    }
    
    handler = handlers.get(option)
    if not handler:
        raise ValueError(f"Unknown option: {option}")
    
    return handler()
```

### Code Duplication Detection
```python
# BAD: Code duplication
class EmailService:
    def send_welcome_email(self, user):
        smtp_server = smtplib.SMTP('smtp.example.com', 587)
        smtp_server.starttls()
        smtp_server.login(self.username, self.password)
        message = f"Welcome {user.name}!"
        smtp_server.send_message(message)
        smtp_server.quit()
    
    def send_password_reset_email(self, user):
        smtp_server = smtplib.SMTP('smtp.example.com', 587)  # Duplicate
        smtp_server.starttls()                              # Duplicate
        smtp_server.login(self.username, self.password)     # Duplicate
        message = f"Reset password for {user.name}"
        smtp_server.send_message(message)                   # Duplicate
        smtp_server.quit()                                  # Duplicate

# GOOD: Extract common functionality
class EmailService:
    def _send_email(self, message: str):
        """Common email sending logic"""
        smtp_server = smtplib.SMTP('smtp.example.com', 587)
        smtp_server.starttls()
        smtp_server.login(self.username, self.password)
        smtp_server.send_message(message)
        smtp_server.quit()
    
    def send_welcome_email(self, user):
        message = f"Welcome {user.name}!"
        self._send_email(message)
    
    def send_password_reset_email(self, user):
        message = f"Reset password for {user.name}"
        self._send_email(message)
```

### Error Handling Review
```python
# BAD: Bare except clauses
def risky_operation():
    try:
        # Some operation
        pass
    except:  # Never catch all exceptions!
        pass

# BAD: Generic exception handling
def file_operation(filename):
    try:
        with open(filename) as f:
            return f.read()
    except Exception:  # Too broad
        return None

# GOOD: Specific exception handling
def file_operation(filename: str) -> str:
    try:
        with open(filename) as f:
            return f.read()
    except FileNotFoundError:
        logger.error(f"File not found: {filename}")
        raise
    except PermissionError:
        logger.error(f"Permission denied: {filename}")
        raise
    except OSError as e:
        logger.error(f"OS error reading {filename}: {e}")
        raise
```

## Dependency Security Review

### Common Vulnerability Patterns
```python
# Check for these security issues:

# 1. Outdated dependencies
# Review requirements.txt/pyproject.toml for:
# - Known vulnerable packages
# - Very old versions
# - Packages with security advisories

# 2. Unsafe deserialization
import pickle  # RED FLAG for untrusted data
def load_user_data(data):
    return pickle.loads(data)  # DANGEROUS!

# Use safe alternatives:
import json
def load_user_data_safe(data: str) -> dict:
    return json.loads(data)

# 3. Path traversal vulnerabilities
import os
def read_config_file(filename):
    # BAD: No validation
    path = os.path.join('/config', filename)
    with open(path) as f:
        return f.read()

def read_config_file_safe(filename: str) -> str:
    # GOOD: Validate and sanitize
    from pathlib import Path
    config_dir = Path('/config').resolve()
    file_path = (config_dir / filename).resolve()
    
    # Ensure file is within config directory
    if not str(file_path).startswith(str(config_dir)):
        raise ValueError("Invalid file path")
    
    with file_path.open() as f:
        return f.read()
```

## Review Checklist Template

### Security Checklist
- [ ] Input validation on all user inputs
- [ ] Parameterized queries for database access
- [ ] Proper authentication and authorization
- [ ] Sensitive data not logged or exposed
- [ ] HTTPS/TLS for data transmission
- [ ] Secure password storage (hashed + salt)
- [ ] Rate limiting for APIs
- [ ] No hardcoded secrets in code
- [ ] Dependency security scan results

### Performance Checklist
- [ ] Algorithm complexity analysis (Big O)
- [ ] Database queries optimized (no N+1)
- [ ] Proper use of async/await patterns
- [ ] Memory usage considerations
- [ ] Caching strategy where appropriate
- [ ] Connection pooling for databases
- [ ] Lazy loading for expensive operations
- [ ] Pagination for large data sets

### Maintainability Checklist
- [ ] Code complexity under control (< 10 cyclomatic)
- [ ] DRY principle followed (no code duplication)
- [ ] SOLID principles applied
- [ ] Proper error handling with specific exceptions
- [ ] Comprehensive test coverage (>80%)
- [ ] Clear documentation and docstrings
- [ ] Consistent code formatting (PEP 8)
- [ ] Type hints for public APIs
- [ ] Meaningful variable and function names

### Code Quality Red Flags
- Functions longer than 50 lines
- Classes with more than 10 methods
- Nested if statements more than 3 levels deep
- Catch-all exception handlers
- Magic numbers without constants
- Global variables
- Functions with more than 5 parameters
- Missing docstrings for public functions
- Code commented out instead of removed
- TODO comments older than 30 days

Use this skill as a comprehensive guide for thorough code review covering security, performance, and maintainability aspects.