# Security

## Table of Contents

1. [Authentication & Authorization](#authentication--authorization)
2. [Input Validation & Sanitization](#input-validation--sanitization)
3. [Rate Limiting](#rate-limiting)

---

## Authentication & Authorization

### Password Hashing

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)
```

### JWT Tokens

```python
from datetime import datetime, timedelta
from jose import JWTError, jwt

SECRET_KEY = settings.jwt_secret
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES))
    to_encode.update({"exp": expire, "iat": datetime.utcnow()})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

def decode_access_token(token: str) -> dict:
    try:
        return jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    except JWTError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Could not validate credentials")
```

### Authentication Dependency

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    user_repo: UserRepository = Depends(get_user_repository)
) -> User:
    payload = decode_access_token(credentials.credentials)

    user_id = payload.get("sub")
    if not user_id:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token")

    user = await user_repo.get_by_id(user_id)
    if not user:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="User not found")

    if not user.is_active:
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Inactive user")

    return user
```

### Role-Based Access Control

```python
def require_role(*required_roles: str):
    def role_checker(current_user: User = Depends(get_current_user)) -> User:
        if current_user.role not in required_roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Requires one of roles: {', '.join(required_roles)}"
            )
        return current_user
    return role_checker

@router.post("/admin/users")
async def create_admin_user(
    user_data: UserCreate,
    current_user: User = Depends(require_role("admin"))
):
    pass  # Only admins can access
```

### Permission-Based Access Control

```python
class Permission(Enum):
    USER_READ = "user:read"
    USER_WRITE = "user:write"
    ORDER_READ = "order:read"
    ORDER_WRITE = "order:write"
    ADMIN = "admin"

def require_permission(*required_permissions: Permission):
    def permission_checker(current_user: User = Depends(get_current_user)) -> User:
        user_permissions = set(current_user.permissions)
        required = set(p.value for p in required_permissions)

        if not required.issubset(user_permissions):
            raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Insufficient permissions")
        return current_user
    return permission_checker

@router.delete("/orders/{order_id}")
async def delete_order(
    order_id: UUID,
    current_user: User = Depends(require_permission(Permission.ORDER_WRITE))
):
    pass
```

---

## Input Validation & Sanitization

```python
from pydantic import BaseModel, validator, Field
import bleach

class SecureUserCreate(BaseModel):
    email: EmailStr
    name: str = Field(..., min_length=1, max_length=100)
    bio: Optional[str] = Field(None, max_length=1000)

    @validator("name")
    def sanitize_name(cls, v):
        return bleach.clean(v, strip=True)

    @validator("bio")
    def sanitize_bio(cls, v):
        if v is None:
            return v
        allowed_tags = ["b", "i", "u", "a", "p", "br"]
        allowed_attrs = {"a": ["href", "title"]}
        return bleach.clean(v, tags=allowed_tags, attributes=allowed_attrs, strip=True)

    @validator("email")
    def validate_email_domain(cls, v):
        disposable_domains = ["tempmail.com", "throwaway.email"]
        domain = v.split("@")[1]
        if domain in disposable_domains:
            raise ValueError("Disposable email addresses are not allowed")
        return v

# SQL Injection Prevention (use parameterized queries)
from sqlalchemy import text

async def safe_raw_query(session: AsyncSession, user_input: str):
    # ✅ GOOD: Parameterized query
    result = await session.execute(
        text("SELECT * FROM users WHERE email = :email"),
        {"email": user_input}
    )

    # ❌ BAD: String interpolation (SQL injection vulnerability)
    # result = await session.execute(text(f"SELECT * FROM users WHERE email = '{user_input}'"))
```

---

## Rate Limiting

### Using slowapi

```python
from fastapi import Request
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)

app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@router.post("/login")
@limiter.limit("5/minute")
async def login(request: Request, credentials: LoginCredentials):
    pass

@router.get("/api/data")
@limiter.limit("100/hour")
async def get_data(request: Request):
    pass
```

### Custom Redis Rate Limiter

```python
class RedisRateLimiter:
    def __init__(self, redis: Redis):
        self._redis = redis

    async def is_allowed(self, key: str, max_requests: int, window_seconds: int) -> bool:
        current = await self._redis.incr(key)

        if current == 1:
            await self._redis.expire(key, window_seconds)

        return current <= max_requests

async def rate_limit_dependency(
    request: Request,
    limiter: RedisRateLimiter = Depends(get_redis_limiter)
):
    client_ip = request.client.host
    key = f"rate_limit:{client_ip}"

    allowed = await limiter.is_allowed(key, max_requests=100, window_seconds=3600)

    if not allowed:
        raise HTTPException(
            status_code=status.HTTP_429_TOO_MANY_REQUESTS,
            detail="Rate limit exceeded"
        )
```
