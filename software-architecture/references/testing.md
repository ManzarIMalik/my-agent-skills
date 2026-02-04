# Testing Strategy

## Table of Contents

1. [Unit Testing](#unit-testing)
2. [Integration Testing](#integration-testing)
3. [End-to-End Testing](#end-to-end-testing)
4. [Test Fixtures & Factories](#test-fixtures--factories)

---

## Unit Testing

```python
import pytest
from unittest.mock import AsyncMock, MagicMock
from decimal import Decimal

# Test domain logic
class TestOrder:
    def test_create_order_success(self):
        items = [OrderItem.create(product_id=uuid4(), quantity=2, price=Decimal("10.00"))]
        order = Order.create(user_id=uuid4(), items=items)

        assert order.status == OrderStatus.PENDING
        assert order.subtotal == Decimal("20.00")

    def test_create_order_without_items_fails(self):
        with pytest.raises(BusinessRuleViolationError) as exc_info:
            Order.create(user_id=uuid4(), items=[])

        assert exc_info.value.rule == "order_must_have_items"

    def test_confirm_order_success(self):
        order = create_order_fixture()
        order.confirm()

        assert order.status == OrderStatus.CONFIRMED

    def test_confirm_non_pending_order_fails(self):
        order = create_order_fixture()
        order.confirm()

        with pytest.raises(BusinessRuleViolationError):
            order.confirm()

# Test use cases with mocks
class TestGetUserUseCase:
    @pytest.fixture
    def use_case(self):
        return GetUserUseCase(
            user_repo=AsyncMock(spec=UserRepository),
            cache=AsyncMock(spec=CacheService)
        )

    @pytest.mark.asyncio
    async def test_returns_cached_user(self, use_case):
        user_data = '{"id": "123", "email": "test@example.com", "name": "Test"}'
        use_case.cache.get.return_value = user_data

        result = await use_case.execute("123")

        assert result.email == "test@example.com"
        use_case.user_repo.get_by_id.assert_not_called()

    @pytest.mark.asyncio
    async def test_fetches_from_db_on_cache_miss(self, use_case):
        use_case.cache.get.return_value = None
        use_case.user_repo.get_by_id.return_value = User(
            id="123", email=Email("test@example.com"), name="Test"
        )

        result = await use_case.execute("123")

        assert result.email == "test@example.com"
        use_case.cache.set.assert_called_once()

    @pytest.mark.asyncio
    async def test_raises_not_found_error(self, use_case):
        use_case.cache.get.return_value = None
        use_case.user_repo.get_by_id.return_value = None

        with pytest.raises(EntityNotFoundError):
            await use_case.execute("123")

# Test value objects
class TestEmail:
    def test_valid_email(self):
        email = Email("test@example.com")
        assert str(email) == "test@example.com"

    def test_invalid_email_raises_error(self):
        with pytest.raises(ValueError):
            Email("invalid-email")
```

---

## Integration Testing

```python
import pytest
from testcontainers.postgres import PostgresContainer
from testcontainers.redis import RedisContainer
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:15") as postgres:
        yield postgres

@pytest.fixture(scope="session")
def redis_container():
    with RedisContainer("redis:7") as redis:
        yield redis

@pytest.fixture
async def db_session(postgres_container):
    url = postgres_container.get_connection_url().replace("postgresql://", "postgresql+asyncpg://")
    engine = create_async_engine(url)

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    async_session = async_sessionmaker(engine, expire_on_commit=False)

    async with async_session() as session:
        yield session
        await session.rollback()

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

@pytest.fixture
async def redis_client(redis_container):
    from redis.asyncio import Redis
    client = Redis.from_url(redis_container.get_connection_url())
    yield client
    await client.flushall()
    await client.close()

# Repository tests
class TestPostgresUserRepository:
    @pytest.mark.asyncio
    async def test_save_and_get_user(self, db_session):
        repo = PostgresUserRepository(db_session)

        user = User(email=Email("test@example.com"), name="Test User")
        saved = await repo.save(user)

        fetched = await repo.get_by_id(str(saved.id))

        assert fetched is not None
        assert fetched.email.value == "test@example.com"

    @pytest.mark.asyncio
    async def test_get_by_email(self, db_session):
        repo = PostgresUserRepository(db_session)

        user = User(email=Email("unique@example.com"), name="Test")
        await repo.save(user)

        fetched = await repo.get_by_email("unique@example.com")

        assert fetched is not None
        assert str(fetched.id) == str(user.id)

# Cache tests
class TestRedisCacheService:
    @pytest.mark.asyncio
    async def test_set_and_get(self, redis_client):
        cache = RedisCacheService(redis_client)

        await cache.set("key", "value", ttl=60)
        result = await cache.get("key")

        assert result == "value"

    @pytest.mark.asyncio
    async def test_expired_key_returns_none(self, redis_client):
        cache = RedisCacheService(redis_client)

        await cache.set("key", "value", ttl=1)
        await asyncio.sleep(1.1)

        result = await cache.get("key")
        assert result is None
```

---

## End-to-End Testing

```python
import pytest
from httpx import AsyncClient
from asgi_lifespan import LifespanManager

@pytest.fixture
async def app():
    from main import app
    async with LifespanManager(app):
        yield app

@pytest.fixture
async def client(app):
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client

@pytest.fixture
async def auth_headers(client):
    # Create test user and get token
    response = await client.post("/api/v1/auth/register", json={
        "email": "test@example.com",
        "name": "Test User",
        "password": "SecurePass123"
    })
    token = response.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}

class TestUserAPI:
    @pytest.mark.asyncio
    async def test_create_user(self, client):
        response = await client.post("/api/v1/users", json={
            "email": "new@example.com",
            "name": "New User",
            "password": "SecurePass123"
        })

        assert response.status_code == 201
        assert response.json()["email"] == "new@example.com"

    @pytest.mark.asyncio
    async def test_get_user_unauthorized(self, client):
        response = await client.get("/api/v1/users/123")
        assert response.status_code == 401

    @pytest.mark.asyncio
    async def test_get_user_success(self, client, auth_headers):
        # First create
        create_response = await client.post("/api/v1/users", json={
            "email": "fetch@example.com", "name": "Fetch User", "password": "SecurePass123"
        })
        user_id = create_response.json()["id"]

        # Then fetch
        response = await client.get(f"/api/v1/users/{user_id}", headers=auth_headers)

        assert response.status_code == 200
        assert response.json()["email"] == "fetch@example.com"

class TestOrderAPI:
    @pytest.mark.asyncio
    async def test_create_order_flow(self, client, auth_headers):
        # Create order
        response = await client.post("/api/v1/orders", headers=auth_headers, json={
            "items": [{"product_id": str(uuid4()), "quantity": 2, "price": "25.00"}]
        })

        assert response.status_code == 201
        order_id = response.json()["id"]

        # Confirm order
        response = await client.post(f"/api/v1/orders/{order_id}/confirm", headers=auth_headers)
        assert response.status_code == 200
        assert response.json()["status"] == "confirmed"
```

---

## Test Fixtures & Factories

```python
import factory
from factory.alchemy import SQLAlchemyModelFactory

# Factories
class UserFactory(SQLAlchemyModelFactory):
    class Meta:
        model = UserModel
        sqlalchemy_session_persistence = "commit"

    id = factory.LazyFunction(uuid4)
    email = factory.Sequence(lambda n: f"user{n}@example.com")
    name = factory.Faker("name")
    password_hash = factory.LazyFunction(lambda: hash_password("password123"))
    tier = UserTierEnum.BRONZE
    is_active = True

class OrderFactory(SQLAlchemyModelFactory):
    class Meta:
        model = OrderModel
        sqlalchemy_session_persistence = "commit"

    id = factory.LazyFunction(uuid4)
    user = factory.SubFactory(UserFactory)
    status = OrderStatusEnum.PENDING
    total = factory.LazyFunction(lambda: Decimal("100.00"))

# Fixtures
@pytest.fixture
def user_factory(db_session):
    UserFactory._meta.sqlalchemy_session = db_session
    return UserFactory

@pytest.fixture
def order_factory(db_session):
    OrderFactory._meta.sqlalchemy_session = db_session
    return OrderFactory

# Usage
class TestOrderRepository:
    @pytest.mark.asyncio
    async def test_find_by_user(self, db_session, user_factory, order_factory):
        user = user_factory()
        order_factory.create_batch(3, user=user)
        order_factory.create_batch(2)  # Other user's orders

        repo = PostgresOrderRepository(db_session)
        orders = await repo.find_by_user(user.id)

        assert len(orders) == 3

# Domain fixtures
def create_user_fixture(**kwargs) -> User:
    defaults = {
        "email": Email("test@example.com"),
        "name": "Test User",
        "tier": UserTier.BRONZE,
        "is_active": True
    }
    defaults.update(kwargs)
    return User(**defaults)

def create_order_fixture(**kwargs) -> Order:
    items = kwargs.pop("items", [
        OrderItem.create(product_id=uuid4(), quantity=1, price=Decimal("50.00"))
    ])
    defaults = {"user_id": uuid4(), "items": items}
    defaults.update(kwargs)
    return Order.create(**defaults)
```
