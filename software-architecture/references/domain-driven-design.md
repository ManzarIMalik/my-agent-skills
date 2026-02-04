# Domain-Driven Design

## Table of Contents

1. [Entities](#entities)
2. [Value Objects](#value-objects)
3. [Domain Events](#domain-events)
4. [Aggregates](#aggregates)

---

## Entities

Entities have identity and lifecycle. Two entities are equal if they have the same ID.

```python
from dataclasses import dataclass, field
from datetime import datetime
from uuid import UUID, uuid4

@dataclass
class Entity:
    """Base entity with identity."""
    id: UUID = field(default_factory=uuid4)
    created_at: datetime = field(default_factory=datetime.utcnow)
    updated_at: datetime = field(default_factory=datetime.utcnow)

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Entity):
            return False
        return self.id == other.id

    def __hash__(self) -> int:
        return hash(self.id)

@dataclass
class User(Entity):
    """User aggregate root."""
    email: Email
    name: str
    tier: UserTier = UserTier.BRONZE
    is_active: bool = True

    def upgrade_tier(self) -> None:
        if self.tier == UserTier.BRONZE:
            self.tier = UserTier.SILVER
        elif self.tier == UserTier.SILVER:
            self.tier = UserTier.GOLD

    def deactivate(self) -> None:
        self.is_active = False
        self.updated_at = datetime.utcnow()

@dataclass
class Order(Entity):
    """Order aggregate root."""
    user_id: UUID
    items: list["OrderItem"]
    status: OrderStatus = OrderStatus.PENDING
    discount: Optional[Discount] = None

    @property
    def subtotal(self) -> Decimal:
        return sum(item.total for item in self.items)

    @property
    def total(self) -> Decimal:
        subtotal = self.subtotal
        return subtotal - self.discount.amount if self.discount else subtotal

    def add_item(self, item: "OrderItem") -> None:
        if self.status != OrderStatus.PENDING:
            raise BusinessRuleViolationError(
                "add_item_to_pending_order",
                f"Cannot add item to order with status {self.status}"
            )
        self.items.append(item)
        self.updated_at = datetime.utcnow()

    def confirm(self) -> None:
        if self.status != OrderStatus.PENDING:
            raise BusinessRuleViolationError(
                "confirm_pending_order",
                f"Cannot confirm order with status {self.status}"
            )
        self.status = OrderStatus.CONFIRMED
        self.updated_at = datetime.utcnow()

    def cancel(self) -> None:
        if self.status in [OrderStatus.SHIPPED, OrderStatus.DELIVERED]:
            raise BusinessRuleViolationError(
                "cancel_unshipped_order",
                f"Cannot cancel order with status {self.status}"
            )
        self.status = OrderStatus.CANCELLED
        self.updated_at = datetime.utcnow()

    @classmethod
    def create(cls, user_id: UUID, items: list["OrderItem"], discount: Optional[Discount] = None) -> "Order":
        if not items:
            raise BusinessRuleViolationError(
                "order_must_have_items",
                "Order must have at least one item"
            )
        return cls(user_id=user_id, items=items, discount=discount, status=OrderStatus.PENDING)
```

---

## Value Objects

Value objects are immutable and compared by value, not identity.

```python
from dataclasses import dataclass
from decimal import Decimal
import re

@dataclass(frozen=True)
class ValueObject:
    """Base value object - immutable and compared by value."""
    pass

@dataclass(frozen=True)
class Email(ValueObject):
    """Email value object with validation."""
    value: str

    def __post_init__(self):
        if not self._is_valid(self.value):
            raise ValueError(f"Invalid email address: {self.value}")

    @staticmethod
    def _is_valid(email: str) -> bool:
        pattern = r'^[\w\.-]+@[\w\.-]+\.\w+$'
        return bool(re.match(pattern, email))

    def __str__(self) -> str:
        return self.value

@dataclass(frozen=True)
class Money(ValueObject):
    """Money value object with currency."""
    amount: Decimal
    currency: str = "USD"

    def __post_init__(self):
        if self.amount < 0:
            raise ValueError("Money amount cannot be negative")
        if not self.currency or len(self.currency) != 3:
            raise ValueError(f"Invalid currency code: {self.currency}")

    def add(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError(f"Cannot add {self.currency} and {other.currency}")
        return Money(self.amount + other.amount, self.currency)

    def multiply(self, multiplier: Decimal) -> "Money":
        return Money(self.amount * multiplier, self.currency)

    def __str__(self) -> str:
        return f"{self.amount:.2f} {self.currency}"

@dataclass(frozen=True)
class Address(ValueObject):
    """Address value object."""
    street: str
    city: str
    state: str
    postal_code: str
    country: str

    def __post_init__(self):
        if not all([self.street, self.city, self.state, self.postal_code, self.country]):
            raise ValueError("All address fields are required")

    def __str__(self) -> str:
        return f"{self.street}, {self.city}, {self.state} {self.postal_code}, {self.country}"

@dataclass(frozen=True)
class Discount(ValueObject):
    """Discount value object."""
    amount: Decimal
    reason: str

    def __post_init__(self):
        if self.amount < 0:
            raise ValueError("Discount amount cannot be negative")
```

---

## Domain Events

Events capture something that happened in the domain.

```python
from dataclasses import dataclass, field
from datetime import datetime
from uuid import UUID, uuid4

@dataclass
class DomainEvent:
    """Base domain event."""
    event_id: UUID = field(default_factory=uuid4)
    occurred_at: datetime = field(default_factory=datetime.utcnow)

    @property
    def event_type(self) -> str:
        return self.__class__.__name__

@dataclass
class OrderCreatedEvent(DomainEvent):
    """Published when order is created."""
    order_id: UUID
    user_id: UUID
    total: Decimal
    items_count: int

    @classmethod
    def from_entity(cls, order: Order) -> "OrderCreatedEvent":
        return cls(
            order_id=order.id,
            user_id=order.user_id,
            total=order.total,
            items_count=len(order.items)
        )

@dataclass
class OrderConfirmedEvent(DomainEvent):
    """Published when order is confirmed."""
    order_id: UUID
    user_id: UUID

@dataclass
class OrderCancelledEvent(DomainEvent):
    """Published when order is cancelled."""
    order_id: UUID
    user_id: UUID
    reason: str

@dataclass
class UserRegisteredEvent(DomainEvent):
    """Published when user registers."""
    user_id: UUID
    email: str
    name: str

@dataclass
class PaymentProcessedEvent(DomainEvent):
    """Published when payment is processed."""
    payment_id: UUID
    order_id: UUID
    amount: Decimal
    status: str
```

---

## Aggregates

Aggregates are clusters of entities and value objects with a root entity that controls access.

```python
@dataclass
class Order(Entity):
    """
    Order is an aggregate root that ensures consistency of its items.
    Only the Order can add/remove items, ensuring business rules are enforced.
    """
    user_id: UUID
    _items: list["OrderItem"] = field(default_factory=list)
    status: OrderStatus = OrderStatus.PENDING
    discount: Optional[Discount] = None
    _events: list[DomainEvent] = field(default_factory=list, init=False)

    @property
    def items(self) -> list["OrderItem"]:
        """Read-only access to items."""
        return self._items.copy()

    def add_item(self, product_id: UUID, quantity: int, price: Decimal) -> None:
        """Add item through aggregate root to maintain invariants."""
        if self.status != OrderStatus.PENDING:
            raise BusinessRuleViolationError(
                "add_item_to_pending_order",
                "Can only add items to pending orders"
            )

        # Business rule: max 10 items per order
        if len(self._items) >= 10:
            raise BusinessRuleViolationError(
                "max_items_per_order",
                "Order cannot have more than 10 items"
            )

        item = OrderItem.create(product_id, quantity, price)
        self._items.append(item)
        self.updated_at = datetime.utcnow()

    def remove_item(self, item_id: UUID) -> None:
        if self.status != OrderStatus.PENDING:
            raise BusinessRuleViolationError(
                "remove_item_from_pending_order",
                "Can only remove items from pending orders"
            )

        self._items = [item for item in self._items if item.id != item_id]
        self.updated_at = datetime.utcnow()

    def confirm(self) -> None:
        """Confirm order and raise domain event."""
        if self.status != OrderStatus.PENDING:
            raise BusinessRuleViolationError(
                "confirm_pending_order",
                f"Cannot confirm order with status {self.status}"
            )

        if not self._items:
            raise BusinessRuleViolationError(
                "confirm_non_empty_order",
                "Cannot confirm order without items"
            )

        self.status = OrderStatus.CONFIRMED
        self.updated_at = datetime.utcnow()

        # Raise domain event
        self._events.append(OrderConfirmedEvent(order_id=self.id, user_id=self.user_id))

    def collect_events(self) -> list[DomainEvent]:
        """Collect and clear domain events."""
        events = self._events.copy()
        self._events.clear()
        return events

@dataclass
class OrderItem(Entity):
    """
    Order item is part of the Order aggregate.
    It cannot exist without an Order and should not be accessed directly.
    """
    product_id: UUID
    quantity: int
    unit_price: Decimal

    @property
    def total(self) -> Decimal:
        return self.unit_price * self.quantity

    @classmethod
    def create(cls, product_id: UUID, quantity: int, price: Decimal) -> "OrderItem":
        if quantity <= 0:
            raise ValueError("Quantity must be positive")
        if price < 0:
            raise ValueError("Price cannot be negative")

        return cls(product_id=product_id, quantity=quantity, unit_price=price)
```
