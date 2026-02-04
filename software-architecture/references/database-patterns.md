# Database Patterns

## Table of Contents

1. [SQLAlchemy Models](#sqlalchemy-models)
2. [Async Session Setup](#async-session-setup)
3. [Query Optimization](#query-optimization)
4. [Alembic Migrations](#alembic-migrations)

---

## SQLAlchemy Models

```python
from sqlalchemy import String, ForeignKey, Numeric, DateTime, Boolean, Enum as SQLEnum, Index
from sqlalchemy.orm import Mapped, mapped_column, relationship, DeclarativeBase
from sqlalchemy.dialects.postgresql import UUID as PGUUID, JSONB
import enum

class Base(DeclarativeBase):
    pass

class UserTierEnum(str, enum.Enum):
    BRONZE = "bronze"
    SILVER = "silver"
    GOLD = "gold"

class UserModel(Base):
    __tablename__ = "users"

    id: Mapped[UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    name: Mapped[str] = mapped_column(String(100))
    password_hash: Mapped[str] = mapped_column(String(255))
    tier: Mapped[UserTierEnum] = mapped_column(SQLEnum(UserTierEnum), default=UserTierEnum.BRONZE)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)

    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow, onupdate=datetime.utcnow)

    orders: Mapped[list["OrderModel"]] = relationship(back_populates="user", cascade="all, delete-orphan")

    __table_args__ = (
        Index("idx_users_email_active", "email", "is_active"),
    )

class OrderStatusEnum(str, enum.Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class OrderModel(Base):
    __tablename__ = "orders"

    id: Mapped[UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True)
    user_id: Mapped[UUID] = mapped_column(PGUUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), index=True)
    status: Mapped[OrderStatusEnum] = mapped_column(SQLEnum(OrderStatusEnum), default=OrderStatusEnum.PENDING, index=True)
    metadata: Mapped[dict] = mapped_column(JSONB, default=dict)
    total: Mapped[Decimal] = mapped_column(Numeric(10, 2))

    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow, onupdate=datetime.utcnow)

    user: Mapped["UserModel"] = relationship(back_populates="orders")
    items: Mapped[list["OrderItemModel"]] = relationship(back_populates="order", cascade="all, delete-orphan", lazy="selectin")

    __table_args__ = (
        Index("idx_orders_user_status", "user_id", "status"),
        Index("idx_orders_created_at", "created_at"),
    )

class OrderItemModel(Base):
    __tablename__ = "order_items"

    id: Mapped[UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True)
    order_id: Mapped[UUID] = mapped_column(PGUUID(as_uuid=True), ForeignKey("orders.id", ondelete="CASCADE"))
    product_id: Mapped[UUID] = mapped_column(PGUUID(as_uuid=True), index=True)
    quantity: Mapped[int] = mapped_column()
    unit_price: Mapped[Decimal] = mapped_column(Numeric(10, 2))

    order: Mapped["OrderModel"] = relationship(back_populates="items")
```

---

## Async Session Setup

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.pool import NullPool, QueuePool

engine = create_async_engine(
    settings.database_url,
    poolclass=QueuePool,
    pool_size=20,
    max_overflow=10,
    pool_timeout=30,
    pool_recycle=3600,
    pool_pre_ping=True,
    echo=settings.debug,
    connect_args={"server_settings": {"jit": "off"}}
)

# For testing
test_engine = create_async_engine(settings.test_database_url, poolclass=NullPool, echo=False)

async_session_maker = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
    autoflush=False,
    autocommit=False,
)

async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()
```

---

## Query Optimization

```python
from sqlalchemy import select, func
from sqlalchemy.orm import selectinload, joinedload

class PostgresOrderRepository(OrderRepository):
    def __init__(self, session: AsyncSession):
        self._session = session

    # Prevent N+1 with selectinload (2 queries: 1 for order, 1 for all items)
    async def get_with_items(self, order_id: UUID) -> Optional[Order]:
        result = await self._session.execute(
            select(OrderModel)
            .options(selectinload(OrderModel.items))
            .where(OrderModel.id == order_id)
        )
        model = result.scalar_one_or_none()
        return self._to_entity(model) if model else None

    # Joined eager loading (single query)
    async def get_with_user(self, order_id: UUID) -> Optional[Order]:
        result = await self._session.execute(
            select(OrderModel)
            .options(joinedload(OrderModel.user))
            .where(OrderModel.id == order_id)
        )
        model = result.scalar_one_or_none()
        return self._to_entity(model) if model else None

    # Efficient pagination
    async def find_paginated(self, page: int, page_size: int, user_id: Optional[UUID] = None) -> tuple[list[Order], int]:
        query = select(OrderModel).options(selectinload(OrderModel.items))
        count_query = select(func.count()).select_from(OrderModel)

        if user_id:
            query = query.where(OrderModel.user_id == user_id)
            count_query = count_query.where(OrderModel.user_id == user_id)

        total = (await self._session.execute(count_query)).scalar()

        query = query.offset((page - 1) * page_size).limit(page_size)
        result = await self._session.execute(query)

        return [self._to_entity(m) for m in result.scalars().all()], total

    # Bulk insert
    async def bulk_create(self, orders: list[Order]) -> list[Order]:
        models = [self._to_model(order) for order in orders]
        self._session.add_all(models)
        await self._session.flush()

        for model in models:
            await self._session.refresh(model)

        return [self._to_entity(m) for m in models]
```

---

## Alembic Migrations

```python
# alembic/env.py
from src.infrastructure.database.models import Base
from src.config import settings

config = context.config
config.set_main_option("sqlalchemy.url", settings.database_url)
target_metadata = Base.metadata

def run_migrations_online():
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            compare_type=True,
            compare_server_default=True,
        )

        with context.begin_transaction():
            context.run_migrations()

# Generate: alembic revision --autogenerate -m "Add order status index"

# Migration file example
def upgrade():
    op.create_index('idx_orders_status_created', 'orders', ['status', 'created_at'], postgresql_using='btree')

def downgrade():
    op.drop_index('idx_orders_status_created', 'orders')
```
