# Architecture Patterns

## Table of Contents

1. [Dependency Injection](#dependency-injection)
2. [Repository Pattern](#repository-pattern)
3. [Unit of Work Pattern](#unit-of-work-pattern)
4. [Service Layer Pattern](#service-layer-pattern)

---

## Dependency Injection

### Define Interface (Port)

```python
from abc import ABC, abstractmethod
from typing import Protocol

# Using ABC
class UserRepository(ABC):
    @abstractmethod
    async def get_by_id(self, user_id: str) -> User | None:
        pass

    @abstractmethod
    async def get_by_email(self, email: str) -> User | None:
        pass

    @abstractmethod
    async def save(self, user: User) -> User:
        pass

    @abstractmethod
    async def delete(self, user_id: str) -> None:
        pass

# Or using Protocol (structural subtyping)
class UserRepository(Protocol):
    async def get_by_id(self, user_id: str) -> User | None: ...
    async def save(self, user: User) -> User: ...
```

### Implement Adapter

```python
class PostgresUserRepository(UserRepository):
    def __init__(self, session: AsyncSession):
        self._session = session

    async def get_by_id(self, user_id: str) -> User | None:
        result = await self._session.execute(
            select(UserModel).where(UserModel.id == user_id)
        )
        model = result.scalar_one_or_none()
        return self._to_entity(model) if model else None

    async def save(self, user: User) -> User:
        model = self._to_model(user)
        self._session.add(model)
        await self._session.flush()
        return user

    def _to_entity(self, model: UserModel) -> User:
        return User(
            id=model.id,
            email=Email(model.email),
            name=model.name,
            created_at=model.created_at
        )

    def _to_model(self, entity: User) -> UserModel:
        return UserModel(
            id=entity.id,
            email=entity.email.value,
            name=entity.name,
            created_at=entity.created_at
        )
```

### Use Case with Injected Dependencies

```python
@dataclass
class GetUserUseCase:
    user_repo: UserRepository
    cache: CacheService

    async def execute(self, user_id: str) -> UserDTO:
        # Try cache first
        cached = await self.cache.get(f"user:{user_id}")
        if cached:
            return UserDTO.parse_raw(cached)

        # Fetch from repository
        user = await self.user_repo.get_by_id(user_id)
        if not user:
            raise EntityNotFoundError("User", user_id)

        # Cache for next time
        dto = UserDTO.from_entity(user)
        await self.cache.set(f"user:{user_id}", dto.json(), ttl=300)

        return dto
```

---

## Repository Pattern

### Generic Repository Interface

```python
from abc import ABC, abstractmethod
from typing import Generic, TypeVar, Optional
from uuid import UUID

T = TypeVar("T")

class Repository(ABC, Generic[T]):
    """Generic repository interface."""

    @abstractmethod
    async def get(self, id: UUID) -> Optional[T]:
        pass

    @abstractmethod
    async def find(self, **criteria) -> list[T]:
        pass

    @abstractmethod
    async def add(self, entity: T) -> T:
        pass

    @abstractmethod
    async def update(self, entity: T) -> T:
        pass

    @abstractmethod
    async def delete(self, id: UUID) -> None:
        pass

    @abstractmethod
    async def exists(self, id: UUID) -> bool:
        pass
```

### Domain-Specific Repository

```python
class OrderRepository(Repository[Order]):
    """Order-specific repository operations."""

    @abstractmethod
    async def find_by_user(self, user_id: UUID) -> list[Order]:
        pass

    @abstractmethod
    async def find_pending(self) -> list[Order]:
        pass

    @abstractmethod
    async def find_by_status(self, status: OrderStatus) -> list[Order]:
        pass
```

### Concrete Implementation

```python
class PostgresOrderRepository(OrderRepository):
    def __init__(self, session: AsyncSession):
        self._session = session

    async def get(self, id: UUID) -> Optional[Order]:
        result = await self._session.execute(
            select(OrderModel)
            .options(selectinload(OrderModel.items))  # Prevent N+1
            .where(OrderModel.id == id)
        )
        model = result.scalar_one_or_none()
        return self._to_entity(model) if model else None

    async def find_by_user(self, user_id: UUID) -> list[Order]:
        result = await self._session.execute(
            select(OrderModel)
            .options(selectinload(OrderModel.items))
            .where(OrderModel.user_id == user_id)
            .order_by(OrderModel.created_at.desc())
        )
        return [self._to_entity(m) for m in result.scalars().all()]

    async def add(self, entity: Order) -> Order:
        model = self._to_model(entity)
        self._session.add(model)
        await self._session.flush()
        await self._session.refresh(model)
        return self._to_entity(model)
```

---

## Unit of Work Pattern

### Interface

```python
class UnitOfWork(ABC):
    """Unit of Work manages transactions across multiple repositories."""

    users: UserRepository
    orders: OrderRepository
    payments: PaymentRepository

    @abstractmethod
    async def __aenter__(self) -> "UnitOfWork":
        pass

    @abstractmethod
    async def __aexit__(self, *args) -> None:
        pass

    @abstractmethod
    async def commit(self) -> None:
        pass

    @abstractmethod
    async def rollback(self) -> None:
        pass
```

### Implementation

```python
class SQLAlchemyUnitOfWork(UnitOfWork):
    def __init__(self, session_factory):
        self._session_factory = session_factory
        self._session: AsyncSession | None = None

    async def __aenter__(self) -> "SQLAlchemyUnitOfWork":
        self._session = self._session_factory()
        self.users = PostgresUserRepository(self._session)
        self.orders = PostgresOrderRepository(self._session)
        self.payments = PostgresPaymentRepository(self._session)
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None:
            await self.rollback()
        await self._session.close()

    async def commit(self) -> None:
        await self._session.commit()

    async def rollback(self) -> None:
        await self._session.rollback()
```

### Usage

```python
@dataclass
class CreateOrderUseCase:
    uow_factory: Callable[[], UnitOfWork]
    event_bus: EventBus

    async def execute(self, command: CreateOrderCommand) -> OrderDTO:
        async with self.uow_factory() as uow:
            # Validate user exists
            user = await uow.users.get(command.user_id)
            if not user:
                raise EntityNotFoundError("User", str(command.user_id))

            # Create order
            order = Order.create(user_id=command.user_id, items=command.items)
            saved_order = await uow.orders.add(order)

            # Process payment
            payment = Payment.create(order_id=saved_order.id, amount=saved_order.total)
            await uow.payments.add(payment)

            # Commit transaction
            await uow.commit()

            # Publish events (after commit)
            await self.event_bus.publish(OrderCreatedEvent.from_entity(saved_order))

            return OrderDTO.from_entity(saved_order)
```

---

## Service Layer Pattern

### Domain Service

```python
class PricingService:
    """Domain service for pricing calculations."""

    def calculate_discount(
        self,
        user: User,
        items: list[OrderItem],
        promo_code: Optional[str] = None
    ) -> Discount:
        base_total = sum(item.price * item.quantity for item in items)

        # User tier discount
        tier_discount = self._calculate_tier_discount(user.tier, base_total)

        # Promo code discount
        promo_discount = Decimal("0")
        if promo_code:
            promo_discount = self._calculate_promo_discount(promo_code, base_total)

        # Take the better discount
        discount_amount = max(tier_discount, promo_discount)

        return Discount(
            amount=discount_amount,
            reason="tier" if tier_discount > promo_discount else "promo"
        )

    def _calculate_tier_discount(self, tier: UserTier, total: Decimal) -> Decimal:
        rates = {
            UserTier.BRONZE: Decimal("0.05"),
            UserTier.SILVER: Decimal("0.10"),
            UserTier.GOLD: Decimal("0.15"),
        }
        return total * rates.get(tier, Decimal("0"))
```

### Application Service

```python
@dataclass
class OrderApplicationService:
    """Application service for order operations."""

    order_repo: OrderRepository
    inventory_service: InventoryService
    pricing_service: PricingService
    notification_service: NotificationService
    event_bus: EventBus

    async def create_order(self, command: CreateOrderCommand) -> OrderDTO:
        # Check inventory
        for item in command.items:
            available = await self.inventory_service.check_availability(
                item.product_id, item.quantity
            )
            if not available:
                raise InsufficientInventoryError(
                    item.product_id, item.quantity,
                    await self.inventory_service.get_quantity(item.product_id)
                )

        # Calculate pricing
        discount = self.pricing_service.calculate_discount(
            user=command.user, items=command.items, promo_code=command.promo_code
        )

        # Create and save order
        order = Order.create(user_id=command.user_id, items=command.items, discount=discount)
        await self.inventory_service.reserve(order.items)
        saved_order = await self.order_repo.add(order)

        # Notifications and events
        await self.notification_service.send_order_confirmation(saved_order)
        await self.event_bus.publish(OrderCreatedEvent.from_entity(saved_order))

        return OrderDTO.from_entity(saved_order)
```
