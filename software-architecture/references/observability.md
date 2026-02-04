# Observability

## Table of Contents

1. [Structured Logging](#structured-logging)
2. [Metrics with Prometheus](#metrics-with-prometheus)
3. [Distributed Tracing](#distributed-tracing)
4. [Health Checks](#health-checks)

---

## Structured Logging

```python
import structlog
from structlog.stdlib import LoggerFactory

# Configure structlog
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.processors.JSONRenderer()
    ],
    wrapper_class=structlog.stdlib.BoundLogger,
    context_class=dict,
    logger_factory=LoggerFactory(),
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()

# Usage with context
async def process_order(order_id: UUID, user_id: UUID):
    log = logger.bind(order_id=str(order_id), user_id=str(user_id))

    log.info("Processing order")

    try:
        result = await do_processing()
        log.info("Order processed successfully", total=result.total)
    except Exception as e:
        log.error("Order processing failed", error=str(e), exc_info=True)
        raise

# Request context middleware
from contextvars import ContextVar

request_id_var: ContextVar[str] = ContextVar("request_id", default="")

@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    request_id = request.headers.get("X-Request-ID", str(uuid4()))
    request_id_var.set(request_id)

    structlog.contextvars.clear_contextvars()
    structlog.contextvars.bind_contextvars(
        request_id=request_id,
        path=request.url.path,
        method=request.method
    )

    start_time = time.time()
    response = await call_next(request)
    duration = time.time() - start_time

    logger.info(
        "Request completed",
        status_code=response.status_code,
        duration_ms=duration * 1000
    )

    response.headers["X-Request-ID"] = request_id
    return response
```

---

## Metrics with Prometheus

```python
from prometheus_client import Counter, Histogram, Gauge, generate_latest, CONTENT_TYPE_LATEST
from fastapi import Response

# Define metrics
REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status_code"]
)

REQUEST_LATENCY = Histogram(
    "http_request_duration_seconds",
    "HTTP request latency",
    ["method", "endpoint"],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

ACTIVE_REQUESTS = Gauge(
    "http_requests_active",
    "Number of active HTTP requests"
)

DB_QUERY_LATENCY = Histogram(
    "db_query_duration_seconds",
    "Database query latency",
    ["query_type"]
)

CACHE_HITS = Counter("cache_hits_total", "Cache hit count", ["cache_name"])
CACHE_MISSES = Counter("cache_misses_total", "Cache miss count", ["cache_name"])

# Middleware
@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    ACTIVE_REQUESTS.inc()
    start_time = time.time()

    try:
        response = await call_next(request)

        REQUEST_COUNT.labels(
            method=request.method,
            endpoint=request.url.path,
            status_code=response.status_code
        ).inc()

        REQUEST_LATENCY.labels(
            method=request.method,
            endpoint=request.url.path
        ).observe(time.time() - start_time)

        return response
    finally:
        ACTIVE_REQUESTS.dec()

# Metrics endpoint
@app.get("/metrics")
async def metrics():
    return Response(content=generate_latest(), media_type=CONTENT_TYPE_LATEST)

# Usage in code
async def get_user_cached(user_id: str) -> User:
    cached = await cache.get(f"user:{user_id}")
    if cached:
        CACHE_HITS.labels(cache_name="user").inc()
        return User.parse_raw(cached)

    CACHE_MISSES.labels(cache_name="user").inc()
    return await fetch_user_from_db(user_id)
```

---

## Distributed Tracing

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor

# Setup
def setup_tracing(service_name: str, otlp_endpoint: str):
    provider = TracerProvider(resource=Resource.create({"service.name": service_name}))

    exporter = OTLPSpanExporter(endpoint=otlp_endpoint)
    provider.add_span_processor(BatchSpanProcessor(exporter))

    trace.set_tracer_provider(provider)

    # Auto-instrument
    FastAPIInstrumentor.instrument_app(app)
    SQLAlchemyInstrumentor().instrument(engine=engine)
    HTTPXClientInstrumentor().instrument()

tracer = trace.get_tracer(__name__)

# Manual spans
async def process_payment(order_id: UUID, amount: Decimal):
    with tracer.start_as_current_span("process_payment") as span:
        span.set_attribute("order_id", str(order_id))
        span.set_attribute("amount", float(amount))

        try:
            result = await payment_gateway.charge(amount)
            span.set_attribute("payment_id", result.payment_id)
            return result
        except PaymentError as e:
            span.set_status(Status(StatusCode.ERROR, str(e)))
            span.record_exception(e)
            raise

# Context propagation
async def call_user_service(user_id: str):
    with tracer.start_as_current_span("call_user_service") as span:
        headers = {}
        inject(headers)  # Inject trace context into headers

        async with httpx.AsyncClient() as client:
            response = await client.get(
                f"{USER_SERVICE_URL}/users/{user_id}",
                headers=headers
            )
            return response.json()
```

---

## Health Checks

```python
from enum import Enum
from dataclasses import dataclass

class HealthStatus(str, Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    UNHEALTHY = "unhealthy"

@dataclass
class ComponentHealth:
    name: str
    status: HealthStatus
    message: Optional[str] = None
    latency_ms: Optional[float] = None

@dataclass
class HealthReport:
    status: HealthStatus
    components: list[ComponentHealth]
    version: str
    uptime_seconds: float

class HealthChecker:
    def __init__(self):
        self._start_time = time.time()
        self._checks: list[Callable[[], Awaitable[ComponentHealth]]] = []

    def register(self, check: Callable[[], Awaitable[ComponentHealth]]):
        self._checks.append(check)

    async def check(self) -> HealthReport:
        components = await asyncio.gather(*[check() for check in self._checks])

        if any(c.status == HealthStatus.UNHEALTHY for c in components):
            overall = HealthStatus.UNHEALTHY
        elif any(c.status == HealthStatus.DEGRADED for c in components):
            overall = HealthStatus.DEGRADED
        else:
            overall = HealthStatus.HEALTHY

        return HealthReport(
            status=overall,
            components=components,
            version=settings.version,
            uptime_seconds=time.time() - self._start_time
        )

# Component checks
async def check_database() -> ComponentHealth:
    start = time.time()
    try:
        async with async_session_maker() as session:
            await session.execute(text("SELECT 1"))
        return ComponentHealth("database", HealthStatus.HEALTHY, latency_ms=(time.time() - start) * 1000)
    except Exception as e:
        return ComponentHealth("database", HealthStatus.UNHEALTHY, message=str(e))

async def check_redis() -> ComponentHealth:
    start = time.time()
    try:
        await redis.ping()
        return ComponentHealth("redis", HealthStatus.HEALTHY, latency_ms=(time.time() - start) * 1000)
    except Exception as e:
        return ComponentHealth("redis", HealthStatus.DEGRADED, message=str(e))

# Endpoints
health_checker = HealthChecker()
health_checker.register(check_database)
health_checker.register(check_redis)

@app.get("/health")
async def health():
    report = await health_checker.check()
    status_code = 200 if report.status == HealthStatus.HEALTHY else 503
    return JSONResponse(content=asdict(report), status_code=status_code)

@app.get("/health/live")
async def liveness():
    return {"status": "ok"}

@app.get("/health/ready")
async def readiness():
    report = await health_checker.check()
    if report.status == HealthStatus.UNHEALTHY:
        raise HTTPException(status_code=503, detail="Not ready")
    return {"status": "ready"}
```
