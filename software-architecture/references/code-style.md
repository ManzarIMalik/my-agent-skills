# Code Style & Standards

## Table of Contents

1. [General Principles](#general-principles)
2. [Naming Conventions](#naming-conventions)
3. [Import Organization](#import-organization)
4. [Error Handling](#error-handling)

---

## General Principles

1. **Early Return Pattern** - Reduce nesting with guard clauses
2. **Type Hints Everywhere** - All public APIs fully typed
3. **Async by Default** - Use async/await for I/O operations
4. **Explicit over Implicit** - No hidden dependencies or magic
5. **Small Functions** - Max 30 lines; extract complex logic
6. **Immutability** - Prefer immutable data structures
7. **Single Responsibility** - One reason to change

---

## Naming Conventions

```python
# Classes: PascalCase
class OrderService: pass
class UserRepository: pass

# Functions/methods: snake_case
async def calculate_order_total(items: list[Item]) -> Decimal: pass
async def send_confirmation_email(user: User) -> None: pass

# Constants: UPPER_SNAKE_CASE
MAX_RETRY_ATTEMPTS = 3
DEFAULT_PAGE_SIZE = 20
API_TIMEOUT_SECONDS = 30

# Private: single underscore prefix
def _validate_email(email: str) -> bool: pass
async def _fetch_from_cache(key: str) -> Any | None: pass

# Type aliases: PascalCase
UserId = str
OrderId = UUID

# Avoid generic names
# ❌ BAD: utils.py, helpers.py, common.py, misc.py, manager.py, handler.py
# ✅ GOOD: email_sender.py, price_calculator.py, order_validator.py
```

---

## Import Organization

```python
# Standard library
import asyncio
import logging
from datetime import datetime, timedelta
from typing import Any, Optional
from uuid import UUID

# Third-party
from fastapi import APIRouter, Depends, HTTPException, status
from pydantic import BaseModel, Field, EmailStr
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

# Local application
from src.domain.entities.user import User
from src.domain.value_objects.email import Email
from src.application.interfaces.user_repository import UserRepository
from src.infrastructure.database.session import get_db_session
```

---

## Error Handling

### Exception Hierarchy

```python
# Domain errors
class DomainError(Exception):
    """Base exception for all domain errors."""
    pass

class EntityNotFoundError(DomainError):
    """Raised when entity is not found."""
    def __init__(self, entity_type: str, entity_id: str):
        self.entity_type = entity_type
        self.entity_id = entity_id
        super().__init__(f"{entity_type} with id {entity_id} not found")

class BusinessRuleViolationError(DomainError):
    """Raised when business rule is violated."""
    def __init__(self, rule: str, reason: str):
        self.rule = rule
        self.reason = reason
        super().__init__(f"Business rule '{rule}' violated: {reason}")

class InsufficientInventoryError(BusinessRuleViolationError):
    """Raised when inventory is insufficient."""
    def __init__(self, product_id: str, requested: int, available: int):
        super().__init__(
            rule="sufficient_inventory",
            reason=f"Product {product_id}: requested {requested}, available {available}"
        )

# Infrastructure errors
class InfrastructureError(Exception):
    """Base for infrastructure errors."""
    pass

class DatabaseConnectionError(InfrastructureError):
    pass

class CacheConnectionError(InfrastructureError):
    pass

class ExternalAPIError(InfrastructureError):
    def __init__(self, service: str, status_code: int, message: str):
        self.service = service
        self.status_code = status_code
        super().__init__(f"{service} API error ({status_code}): {message}")
```

### Proper Exception Handling

```python
async def get_user_by_id(user_id: str) -> UserDTO:
    try:
        user = await user_repo.get_by_id(user_id)
        if not user:
            raise EntityNotFoundError("User", user_id)
        return UserDTO.from_entity(user)
    except DatabaseConnectionError as e:
        logger.error(f"Database connection failed: {e}", exc_info=True)
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail="Database temporarily unavailable"
        )
    except EntityNotFoundError:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"User {user_id} not found"
        )

# Never use bare except
# ❌ BAD
try:
    result = await operation()
except:  # Catches everything, including KeyboardInterrupt
    pass

# ✅ GOOD
try:
    result = await operation()
except OperationError as e:
    logger.error(f"Operation failed: {e}")
    raise
except Exception as e:
    logger.exception(f"Unexpected error: {e}")
    raise
```
