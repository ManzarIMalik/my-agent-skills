# Microservices Patterns

## Table of Contents

1. [Service-to-Service Communication](#service-to-service-communication)
2. [Circuit Breaker Pattern](#circuit-breaker-pattern)
3. [Retry with Exponential Backoff](#retry-with-exponential-backoff)
4. [Service Discovery](#service-discovery)

---

## Service-to-Service Communication

### REST API Calls

```python
import httpx

class UserServiceClient:
    def __init__(self, base_url: str, timeout: int = 30):
        self._client = httpx.AsyncClient(
            base_url=base_url,
            timeout=timeout,
            headers={"User-Agent": "order-service/1.0"}
        )

    async def get_user(self, user_id: UUID) -> User:
        response = await self._client.get(f"/api/v1/users/{user_id}")
        response.raise_for_status()
        return User(**response.json())

    async def check_user_tier(self, user_id: UUID) -> str:
        response = await self._client.get(f"/api/v1/users/{user_id}/tier")
        response.raise_for_status()
        return response.json()["tier"]

    async def close(self) -> None:
        await self._client.aclose()
```

### gRPC Calls

```protobuf
// proto/user_service.proto
syntax = "proto3";
package user;

service UserService {
  rpc GetUser(GetUserRequest) returns (UserResponse);
  rpc CheckUserTier(CheckUserTierRequest) returns (CheckUserTierResponse);
}

message GetUserRequest { string user_id = 1; }
message UserResponse { string id = 1; string email = 2; string name = 3; string tier = 4; }
```

```python
import grpc
from proto import user_service_pb2, user_service_pb2_grpc

class GrpcUserServiceClient:
    def __init__(self, host: str, port: int):
        self._channel = grpc.aio.insecure_channel(f"{host}:{port}")
        self._stub = user_service_pb2_grpc.UserServiceStub(self._channel)

    async def get_user(self, user_id: str) -> User:
        request = user_service_pb2.GetUserRequest(user_id=user_id)
        response = await self._stub.GetUser(request)
        return User(id=UUID(response.id), email=response.email, name=response.name, tier=response.tier)

    async def close(self) -> None:
        await self._channel.close()
```

### Message-Based Communication

```python
class OrderServiceMessaging:
    def __init__(self, event_bus: EventBus):
        self._event_bus = event_bus

    async def publish_order_created(self, order: Order) -> None:
        event = OrderCreatedEvent.from_entity(order)
        await self._event_bus.publish(event)
```

---

## Circuit Breaker Pattern

```python
from enum import Enum
from datetime import datetime, timedelta

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Failing, reject requests
    HALF_OPEN = "half_open"  # Testing recovery

class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: int = 5,
        timeout_duration: int = 60,
        expected_exception: type[Exception] = Exception
    ):
        self.failure_threshold = failure_threshold
        self.timeout_duration = timeout_duration
        self.expected_exception = expected_exception

        self.failure_count = 0
        self.last_failure_time: Optional[datetime] = None
        self.state = CircuitState.CLOSED

    async def call(self, func: Callable[[], Awaitable[T]]) -> T:
        if self.state == CircuitState.OPEN:
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
            else:
                raise CircuitBreakerOpenError("Circuit breaker is OPEN")

        try:
            result = await func()
            self._on_success()
            return result
        except self.expected_exception:
            self._on_failure()
            raise

    def _on_success(self) -> None:
        self.failure_count = 0
        self.state = CircuitState.CLOSED

    def _on_failure(self) -> None:
        self.failure_count += 1
        self.last_failure_time = datetime.utcnow()

        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
            logger.warning(f"Circuit breaker opened after {self.failure_count} failures")

    def _should_attempt_reset(self) -> bool:
        if not self.last_failure_time:
            return True
        return datetime.utcnow() - self.last_failure_time > timedelta(seconds=self.timeout_duration)

# Usage
class ResilientUserServiceClient:
    def __init__(self, client: UserServiceClient):
        self._client = client
        self._circuit_breaker = CircuitBreaker(
            failure_threshold=5, timeout_duration=60, expected_exception=httpx.HTTPError
        )

    async def get_user(self, user_id: UUID) -> User:
        try:
            return await self._circuit_breaker.call(lambda: self._client.get_user(user_id))
        except CircuitBreakerOpenError:
            logger.warning(f"Circuit breaker open, using fallback for user {user_id}")
            return await self._get_user_from_cache(user_id)
```

---

## Retry with Exponential Backoff

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type, before_sleep_log

class ExternalAPIClient:
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=1, max=10),
        retry=retry_if_exception_type((httpx.TimeoutException, httpx.NetworkError)),
        before_sleep=before_sleep_log(logger, logging.WARNING)
    )
    async def fetch_data(self, url: str) -> dict:
        async with httpx.AsyncClient() as client:
            response = await client.get(url, timeout=5.0)
            response.raise_for_status()
            return response.json()

    # Custom retry logic
    async def fetch_with_custom_retry(self, url: str, max_retries: int = 3) -> dict:
        for attempt in range(max_retries):
            try:
                return await self._do_fetch(url)
            except httpx.HTTPError as e:
                if attempt == max_retries - 1:
                    raise

                wait_time = 2 ** attempt  # Exponential backoff
                logger.warning(f"Attempt {attempt + 1} failed: {e}. Retrying in {wait_time}s...")
                await asyncio.sleep(wait_time)
```

---

## Service Discovery

```python
import random

class ServiceRegistry:
    def __init__(self):
        self._services: dict[str, list[str]] = {}

    def register(self, service_name: str, instance_url: str) -> None:
        if service_name not in self._services:
            self._services[service_name] = []

        if instance_url not in self._services[service_name]:
            self._services[service_name].append(instance_url)
            logger.info(f"Registered {service_name} at {instance_url}")

    def deregister(self, service_name: str, instance_url: str) -> None:
        if service_name in self._services:
            self._services[service_name].remove(instance_url)

    def discover(self, service_name: str) -> Optional[str]:
        instances = self._services.get(service_name, [])
        if not instances:
            return None
        return random.choice(instances)  # Simple load balancing

    def get_all_instances(self, service_name: str) -> list[str]:
        return self._services.get(service_name, []).copy()

class ServiceAwareClient:
    def __init__(self, registry: ServiceRegistry, service_name: str):
        self._registry = registry
        self._service_name = service_name

    async def make_request(self, path: str) -> dict:
        instance_url = self._registry.discover(self._service_name)

        if not instance_url:
            raise ServiceUnavailableError(f"{self._service_name} not available")

        async with httpx.AsyncClient() as client:
            response = await client.get(f"{instance_url}{path}")
            response.raise_for_status()
            return response.json()
```
