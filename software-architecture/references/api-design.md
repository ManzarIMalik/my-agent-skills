# API Design

## Table of Contents

1. [FastAPI Router Structure](#fastapi-router-structure)
2. [Request/Response Schemas](#requestresponse-schemas)
3. [Dependency Injection Setup](#dependency-injection-setup)
4. [API Versioning](#api-versioning)

---

## FastAPI Router Structure

```python
# src/presentation/api/v1/users.py
from fastapi import APIRouter, Depends, HTTPException, status, Query

router = APIRouter(prefix="/api/v1/users", tags=["users"])

@router.post(
    "/",
    response_model=UserResponse,
    status_code=status.HTTP_201_CREATED,
    responses={
        201: {"description": "User created successfully"},
        409: {"description": "Email already exists"},
        422: {"description": "Validation error"}
    }
)
async def create_user(
    user_data: UserCreate,
    use_case: CreateUserUseCase = Depends(get_create_user_use_case),
) -> UserResponse:
    """
    Create a new user.

    - **email**: Valid email address (unique)
    - **name**: Full name
    - **password**: Minimum 8 characters
    """
    try:
        user = await use_case.execute(user_data)
        return UserResponse.from_dto(user)
    except EmailAlreadyExistsError as e:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail=f"Email {e.email} is already registered"
        )

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: UUID,
    use_case: GetUserUseCase = Depends(get_get_user_use_case),
) -> UserResponse:
    try:
        user = await use_case.execute(str(user_id))
        return UserResponse.from_dto(user)
    except EntityNotFoundError:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=f"User {user_id} not found")

@router.get("/", response_model=PaginatedResponse[UserResponse])
async def list_users(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    use_case: ListUsersUseCase = Depends(get_list_users_use_case),
) -> PaginatedResponse[UserResponse]:
    result = await use_case.execute(page=page, page_size=page_size)
    return PaginatedResponse(
        items=[UserResponse.from_dto(u) for u in result.items],
        total=result.total, page=page, page_size=page_size
    )

@router.patch("/{user_id}", response_model=UserResponse)
async def update_user(
    user_id: UUID,
    update_data: UserUpdate,
    use_case: UpdateUserUseCase = Depends(get_update_user_use_case),
    current_user: User = Depends(get_current_user),
) -> UserResponse:
    if str(current_user.id) != str(user_id):
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Cannot update other users")

    user = await use_case.execute(str(user_id), update_data)
    return UserResponse.from_dto(user)

@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(
    user_id: UUID,
    use_case: DeleteUserUseCase = Depends(get_delete_user_use_case),
    current_user: User = Depends(get_current_user),
) -> None:
    if str(current_user.id) != str(user_id):
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Cannot delete other users")
    await use_case.execute(str(user_id))
```

---

## Request/Response Schemas

```python
from pydantic import BaseModel, EmailStr, Field, validator
from typing import Generic, TypeVar

class BaseSchema(BaseModel):
    class Config:
        from_attributes = True
        json_encoders = {
            datetime: lambda v: v.isoformat(),
            Decimal: lambda v: str(v)
        }

# Request schemas
class UserCreate(BaseSchema):
    email: EmailStr
    name: str = Field(..., min_length=1, max_length=100)
    password: str = Field(..., min_length=8, max_length=100)

    @validator("password")
    def validate_password(cls, v):
        if not any(c.isupper() for c in v):
            raise ValueError("Must contain at least one uppercase letter")
        if not any(c.isdigit() for c in v):
            raise ValueError("Must contain at least one digit")
        return v

class UserUpdate(BaseSchema):
    name: str | None = Field(None, min_length=1, max_length=100)
    email: EmailStr | None = None

# Response schemas
class UserResponse(BaseSchema):
    id: UUID
    email: str
    name: str
    tier: str
    is_active: bool
    created_at: datetime

    @classmethod
    def from_dto(cls, dto: UserDTO) -> "UserResponse":
        return cls(
            id=dto.id, email=dto.email, name=dto.name,
            tier=dto.tier.value, is_active=dto.is_active, created_at=dto.created_at
        )

# Pagination
T = TypeVar("T")

class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    page_size: int

    @property
    def total_pages(self) -> int:
        return (self.total + self.page_size - 1) // self.page_size

    @property
    def has_next(self) -> bool:
        return self.page < self.total_pages
```

---

## Dependency Injection Setup

```python
# src/presentation/api/dependencies.py
from functools import lru_cache

@lru_cache
def get_settings() -> Settings:
    return Settings()

async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        try:
            yield session
        finally:
            await session.close()

def get_user_repository(session: AsyncSession = Depends(get_db_session)) -> UserRepository:
    return PostgresUserRepository(session)

def get_cache_service(settings: Settings = Depends(get_settings)) -> CacheService:
    return RedisCacheService(settings.redis_url)

def get_create_user_use_case(
    user_repo: UserRepository = Depends(get_user_repository),
    cache: CacheService = Depends(get_cache_service),
    event_bus: EventBus = Depends(get_event_bus),
) -> CreateUserUseCase:
    return CreateUserUseCase(user_repo=user_repo, cache=cache, event_bus=event_bus)

# Authentication
security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    user_repo: UserRepository = Depends(get_user_repository),
) -> User:
    try:
        payload = jwt.decode(credentials.credentials, settings.jwt_secret, algorithms=["HS256"])
        user_id = payload.get("sub")
        if not user_id:
            raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token")
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Token expired")
    except jwt.JWTError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token")

    user = await user_repo.get_by_id(user_id)
    if not user:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="User not found")
    return user

# Role-based access
def require_role(required_role: str):
    async def role_checker(current_user: User = Depends(get_current_user)) -> User:
        if current_user.role != required_role:
            raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail=f"Requires {required_role} role")
        return current_user
    return role_checker
```

---

## API Versioning

```python
# URL versioning (recommended)
from fastapi import APIRouter

v1_router = APIRouter(prefix="/api/v1")
v1_router.include_router(users_router)
v1_router.include_router(orders_router)

v2_router = APIRouter(prefix="/api/v2")
v2_router.include_router(users_router_v2)

# main.py
app = FastAPI()
app.include_router(v1_router)
app.include_router(v2_router)

# Header versioning (alternative)
async def get_api_version(
    api_version: str = Header(default="1", alias="X-API-Version")
) -> str:
    if api_version not in ["1", "2"]:
        raise HTTPException(status_code=400, detail="Unsupported API version")
    return api_version

# Deprecation
@router.get("/old-endpoint")
@deprecated(version="2.0.0", reason="Use /api/v2/new-endpoint", sunset_date="2024-12-31")
async def old_endpoint():
    return {"message": "This endpoint is deprecated"}
```
