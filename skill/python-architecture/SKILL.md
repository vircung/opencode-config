---
name: python-architecture
description: Python system design patterns, project structure, and scalable architecture guidelines
license: MIT
compatibility: opencode
metadata:
  audience: architects
  language: python
  focus: design-patterns
---

# Python Architecture Skill

## Design Patterns for Python

### Creational Patterns

**Factory Pattern**
```python
from abc import ABC, abstractmethod

class Animal(ABC):
    @abstractmethod
    def make_sound(self):
        pass

class Dog(Animal):
    def make_sound(self):
        return "Woof!"

class AnimalFactory:
    _animals = {"dog": Dog}
    
    @classmethod
    def create_animal(cls, animal_type: str) -> Animal:
        if animal_class := cls._animals.get(animal_type):
            return animal_class()
        raise ValueError(f"Unknown animal type: {animal_type}")
```

**Singleton Pattern (Modern Python)**
```python
class DatabaseConnection:
    _instance = None
    _connection = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    
    def get_connection(self):
        if not self._connection:
            self._connection = "database_connection_object"
        return self._connection
```

### Structural Patterns

**Adapter Pattern**
```python
class LegacyPaymentSystem:
    def make_payment(self, amount):
        return f"Legacy payment: ${amount}"

class ModernPaymentInterface:
    def process_payment(self, amount: float) -> str:
        pass

class PaymentAdapter(ModernPaymentInterface):
    def __init__(self, legacy_system: LegacyPaymentSystem):
        self.legacy_system = legacy_system
    
    def process_payment(self, amount: float) -> str:
        return self.legacy_system.make_payment(amount)
```

**Decorator Pattern**
```python
from functools import wraps
import time

def retry(max_attempts: int = 3, delay: float = 1.0):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    time.sleep(delay)
            return wrapper
        return decorator
```

### Behavioral Patterns

**Observer Pattern**
```python
from typing import List, Protocol

class Observer(Protocol):
    def update(self, data: any) -> None:
        ...

class Subject:
    def __init__(self):
        self._observers: List[Observer] = []
        self._state = None
    
    def attach(self, observer: Observer):
        self._observers.append(observer)
    
    def detach(self, observer: Observer):
        self._observers.remove(observer)
    
    def notify(self):
        for observer in self._observers:
            observer.update(self._state)
    
    def set_state(self, state):
        self._state = state
        self.notify()
```

**Strategy Pattern**
```python
from abc import ABC, abstractmethod
from typing import List

class SortStrategy(ABC):
    @abstractmethod
    def sort(self, data: List[int]) -> List[int]:
        pass

class BubbleSort(SortStrategy):
    def sort(self, data: List[int]) -> List[int]:
        # Implementation here
        return sorted(data)  # Simplified

class QuickSort(SortStrategy):
    def sort(self, data: List[int]) -> List[int]:
        # Implementation here
        return sorted(data)  # Simplified

class Sorter:
    def __init__(self, strategy: SortStrategy):
        self._strategy = strategy
    
    def set_strategy(self, strategy: SortStrategy):
        self._strategy = strategy
    
    def sort_data(self, data: List[int]) -> List[int]:
        return self._strategy.sort(data)
```

## Project Structure Patterns

### Small to Medium Projects
```
project_name/
├── src/
│   └── project_name/
│       ├── __init__.py
│       ├── main.py
│       ├── config.py
│       ├── models/
│       │   ├── __init__.py
│       │   └── user.py
│       ├── services/
│       │   ├── __init__.py
│       │   └── user_service.py
│       └── utils/
│           ├── __init__.py
│           └── helpers.py
├── tests/
│   ├── __init__.py
│   ├── test_models/
│   └── test_services/
├── docs/
├── requirements.txt
├── pyproject.toml
└── README.md
```

### Large Projects (Layered Architecture)
```
project_name/
├── src/
│   └── project_name/
│       ├── domain/          # Business logic
│       │   ├── models/
│       │   ├── services/
│       │   └── repositories/
│       ├── infrastructure/  # External concerns
│       │   ├── database/
│       │   ├── api/
│       │   └── messaging/
│       ├── application/     # Use cases
│       │   ├── commands/
│       │   ├── queries/
│       │   └── handlers/
│       └── presentation/    # Controllers, views
│           ├── api/
│           └── cli/
└── ...
```

## SOLID Principles in Python

### Single Responsibility Principle
```python
# BAD: Multiple responsibilities
class User:
    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email
    
    def save_to_database(self):
        # Database logic
        pass
    
    def send_email(self):
        # Email logic
        pass

# GOOD: Single responsibility
class User:
    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email

class UserRepository:
    def save(self, user: User):
        # Database logic
        pass

class EmailService:
    def send_user_email(self, user: User):
        # Email logic
        pass
```

### Dependency Inversion Principle
```python
from abc import ABC, abstractmethod

# Bad: High-level depends on low-level
class EmailService:
    def send_email(self, message: str):
        # SMTP implementation
        pass

class NotificationService:
    def __init__(self):
        self.email_service = EmailService()  # Direct dependency
    
    def notify(self, message: str):
        self.email_service.send_email(message)

# Good: Both depend on abstraction
class MessageSender(ABC):
    @abstractmethod
    def send(self, message: str) -> bool:
        pass

class EmailSender(MessageSender):
    def send(self, message: str) -> bool:
        # SMTP implementation
        return True

class NotificationService:
    def __init__(self, sender: MessageSender):
        self._sender = sender
    
    def notify(self, message: str) -> bool:
        return self._sender.send(message)
```

## Async Architecture Patterns

### Producer-Consumer Pattern
```python
import asyncio
from typing import Any

class AsyncQueue:
    def __init__(self, maxsize: int = 0):
        self._queue = asyncio.Queue(maxsize=maxsize)
    
    async def produce(self, item: Any):
        await self._queue.put(item)
    
    async def consume(self) -> Any:
        return await self._queue.get()
    
    def task_done(self):
        self._queue.task_done()

async def producer(queue: AsyncQueue, items: list):
    for item in items:
        await queue.produce(item)
        await asyncio.sleep(0.1)

async def consumer(queue: AsyncQueue, name: str):
    while True:
        item = await queue.consume()
        print(f"Consumer {name} processing {item}")
        await asyncio.sleep(0.5)
        queue.task_done()
```

### Event-Driven Architecture
```python
from typing import Dict, List, Callable
import asyncio

class EventBus:
    def __init__(self):
        self._handlers: Dict[str, List[Callable]] = {}
    
    def subscribe(self, event_type: str, handler: Callable):
        if event_type not in self._handlers:
            self._handlers[event_type] = []
        self._handlers[event_type].append(handler)
    
    async def publish(self, event_type: str, data: any):
        if handlers := self._handlers.get(event_type, []):
            tasks = [handler(data) for handler in handlers]
            await asyncio.gather(*tasks, return_exceptions=True)

# Usage
async def user_created_handler(user_data):
    print(f"Sending welcome email to {user_data['email']}")

bus = EventBus()
bus.subscribe("user_created", user_created_handler)
await bus.publish("user_created", {"email": "user@example.com"})
```

## API Design Guidelines

### RESTful Resource Design
```python
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from typing import List, Optional

app = FastAPI()

class UserCreate(BaseModel):
    name: str
    email: str

class UserResponse(BaseModel):
    id: int
    name: str
    email: str

class UserUpdate(BaseModel):
    name: Optional[str] = None
    email: Optional[str] = None

# RESTful endpoints
@app.post("/users", response_model=UserResponse, status_code=201)
async def create_user(user: UserCreate):
    # Implementation
    pass

@app.get("/users", response_model=List[UserResponse])
async def list_users(limit: int = 10, offset: int = 0):
    # Implementation
    pass

@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int):
    # Implementation
    pass

@app.patch("/users/{user_id}", response_model=UserResponse)
async def update_user(user_id: int, user_update: UserUpdate):
    # Implementation
    pass

@app.delete("/users/{user_id}", status_code=204)
async def delete_user(user_id: int):
    # Implementation
    pass
```

These patterns provide a solid foundation for building maintainable, scalable Python applications. Choose patterns based on your specific use case and complexity requirements.