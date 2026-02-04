# Event-Driven Architecture

## Table of Contents

1. [Event Bus](#event-bus)
2. [Event Handlers](#event-handlers)
3. [Saga Pattern](#saga-pattern)

---

## Event Bus

### Interface

```python
from abc import ABC, abstractmethod
from typing import Callable, Awaitable

EventHandler = Callable[[DomainEvent], Awaitable[None]]

class EventBus(ABC):
    @abstractmethod
    async def publish(self, event: DomainEvent) -> None:
        pass

    @abstractmethod
    def subscribe(self, event_type: type[DomainEvent], handler: EventHandler) -> None:
        pass
```

### In-Memory Implementation (Development)

```python
class InMemoryEventBus(EventBus):
    def __init__(self):
        self._handlers: dict[type[DomainEvent], list[EventHandler]] = {}

    def subscribe(self, event_type: type[DomainEvent], handler: EventHandler) -> None:
        if event_type not in self._handlers:
            self._handlers[event_type] = []
        self._handlers[event_type].append(handler)

    async def publish(self, event: DomainEvent) -> None:
        handlers = self._handlers.get(type(event), [])
        await asyncio.gather(
            *[handler(event) for handler in handlers],
            return_exceptions=True
        )
```

### RabbitMQ Implementation (Production)

```python
import aio_pika

class RabbitMQEventBus(EventBus):
    def __init__(self, rabbitmq_url: str):
        self._url = rabbitmq_url
        self._connection: Optional[aio_pika.Connection] = None
        self._channel: Optional[aio_pika.Channel] = None
        self._exchange_name = "domain_events"

    async def connect(self) -> None:
        self._connection = await aio_pika.connect_robust(self._url)
        self._channel = await self._connection.channel()
        self._exchange = await self._channel.declare_exchange(
            self._exchange_name, aio_pika.ExchangeType.TOPIC, durable=True
        )

    async def publish(self, event: DomainEvent) -> None:
        if not self._channel:
            await self.connect()

        message = aio_pika.Message(
            body=event.json().encode(),
            content_type="application/json",
            delivery_mode=aio_pika.DeliveryMode.PERSISTENT,
            headers={
                "event_type": event.event_type,
                "event_id": str(event.event_id),
                "occurred_at": event.occurred_at.isoformat()
            }
        )

        await self._exchange.publish(message, routing_key=event.event_type)

    async def subscribe(
        self,
        event_type: type[DomainEvent],
        handler: EventHandler,
        queue_name: Optional[str] = None
    ) -> None:
        if not self._channel:
            await self.connect()

        queue_name = queue_name or f"{event_type.__name__}_queue"
        queue = await self._channel.declare_queue(queue_name, durable=True)
        await queue.bind(self._exchange, routing_key=event_type.__name__)

        async def process_message(message: aio_pika.IncomingMessage):
            async with message.process():
                event_data = json.loads(message.body.decode())
                event = event_type(**event_data)
                await handler(event)

        await queue.consume(process_message)

    async def close(self) -> None:
        if self._connection:
            await self._connection.close()
```

---

## Event Handlers

```python
class SendWelcomeEmailHandler:
    def __init__(self, email_service: EmailService):
        self._email_service = email_service

    async def handle(self, event: UserRegisteredEvent) -> None:
        try:
            await self._email_service.send_welcome_email(to=event.email, name=event.name)
            logger.info(f"Welcome email sent to {event.email}")
        except Exception as e:
            logger.error(f"Failed to send welcome email: {e}")

class UpdateInventoryHandler:
    def __init__(self, inventory_service: InventoryService):
        self._inventory_service = inventory_service

    async def handle(self, event: OrderCreatedEvent) -> None:
        try:
            await self._inventory_service.reserve_for_order(event.order_id)
            logger.info(f"Inventory reserved for order {event.order_id}")
        except InsufficientInventoryError:
            await event_bus.publish(OrderCancelledEvent(
                order_id=event.order_id, reason="insufficient_inventory"
            ))

# Registration
def register_event_handlers(event_bus: EventBus):
    event_bus.subscribe(UserRegisteredEvent, SendWelcomeEmailHandler(email_service).handle)
    event_bus.subscribe(OrderCreatedEvent, UpdateInventoryHandler(inventory_service).handle)
    event_bus.subscribe(OrderCancelledEvent, ReleaseInventoryHandler(inventory_service).handle)
```

---

## Saga Pattern

For distributed transactions across services.

```python
from enum import Enum

class SagaStatus(Enum):
    STARTED = "started"
    COMPLETED = "completed"
    FAILED = "failed"
    COMPENSATING = "compensating"
    COMPENSATED = "compensated"

@dataclass
class SagaStep:
    name: str
    execute: Callable[[], Awaitable[Any]]
    compensate: Callable[[], Awaitable[None]]

class Saga:
    def __init__(self, saga_id: UUID, steps: list[SagaStep]):
        self.saga_id = saga_id
        self.steps = steps
        self.status = SagaStatus.STARTED
        self.completed_steps: list[str] = []
        self.results: dict[str, Any] = {}

    async def execute(self) -> Any:
        try:
            for step in self.steps:
                logger.info(f"Saga {self.saga_id}: Executing {step.name}")
                result = await step.execute()
                self.results[step.name] = result
                self.completed_steps.append(step.name)

            self.status = SagaStatus.COMPLETED
            return self.results
        except Exception as e:
            logger.error(f"Saga {self.saga_id}: Failed at {step.name}: {e}")
            self.status = SagaStatus.FAILED
            await self.compensate()
            raise

    async def compensate(self) -> None:
        self.status = SagaStatus.COMPENSATING

        for step_name in reversed(self.completed_steps):
            step = next(s for s in self.steps if s.name == step_name)
            try:
                logger.info(f"Saga {self.saga_id}: Compensating {step_name}")
                await step.compensate()
            except Exception as e:
                logger.error(f"Saga {self.saga_id}: Compensation failed for {step_name}: {e}")

        self.status = SagaStatus.COMPENSATED

# Example usage
class CreateOrderSaga:
    def __init__(self, order_service, payment_service, inventory_service, notification_service):
        self.order_service = order_service
        self.payment_service = payment_service
        self.inventory_service = inventory_service
        self.notification_service = notification_service

    async def execute(self, command: CreateOrderCommand) -> Order:
        saga_id = uuid4()

        steps = [
            SagaStep(
                name="create_order",
                execute=lambda: self.order_service.create_order(command),
                compensate=lambda: self.order_service.delete_order(saga_id)
            ),
            SagaStep(
                name="reserve_inventory",
                execute=lambda: self.inventory_service.reserve(command.items),
                compensate=lambda: self.inventory_service.release(command.items)
            ),
            SagaStep(
                name="process_payment",
                execute=lambda: self.payment_service.charge(command.payment_info),
                compensate=lambda: self.payment_service.refund(saga_id)
            ),
            SagaStep(
                name="send_confirmation",
                execute=lambda: self.notification_service.send_confirmation(saga_id),
                compensate=lambda: self.notification_service.send_cancellation(saga_id)
            )
        ]

        saga = Saga(saga_id, steps)
        results = await saga.execute()
        return results["create_order"]
```
