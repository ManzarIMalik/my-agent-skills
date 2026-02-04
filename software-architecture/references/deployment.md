# Production Deployment

## Table of Contents

1. [Docker Configuration](#docker-configuration)
2. [Kubernetes Deployment](#kubernetes-deployment)
3. [Configuration Management](#configuration-management)
4. [Production Checklist](#production-checklist)

---

## Docker Configuration

### Multi-Stage Dockerfile

```dockerfile
# Build stage
FROM python:3.11-slim as builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY pyproject.toml poetry.lock ./
RUN pip install poetry && \
    poetry config virtualenvs.create false && \
    poetry install --no-dev --no-interaction --no-ansi

# Production stage
FROM python:3.11-slim as production

WORKDIR /app

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Copy dependencies from builder
COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin

# Copy application code
COPY src/ ./src/
COPY alembic/ ./alembic/
COPY alembic.ini ./

# Set ownership
RUN chown -R appuser:appuser /app

USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health/live || exit 1

EXPOSE 8000

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Docker Compose

```yaml
version: "3.8"

services:
  api:
    build:
      context: .
      target: production
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://user:pass@postgres:5432/app
      - REDIS_URL=redis://redis:6379/0
      - RABBITMQ_URL=amqp://user:pass@rabbitmq:5672/
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: "1"
          memory: 512M

  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: app
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d app"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  rabbitmq:
    image: rabbitmq:3-management
    environment:
      RABBITMQ_DEFAULT_USER: user
      RABBITMQ_DEFAULT_PASS: pass

volumes:
  postgres_data:
```

---

## Kubernetes Deployment

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels:
    app: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: myregistry/api:v1.0.0
          ports:
            - containerPort: 8000
          envFrom:
            - configMapRef:
                name: api-config
            - secretRef:
                name: api-secrets
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 5
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
      terminationGracePeriodSeconds: 30
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8000
  type: ClusterIP
```

### ConfigMap & Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  LOG_LEVEL: "INFO"
  ENVIRONMENT: "production"
  WORKERS: "4"
---
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
type: Opaque
stringData:
  DATABASE_URL: "postgresql+asyncpg://user:pass@postgres:5432/app"
  JWT_SECRET: "your-secret-key"
```

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

---

## Configuration Management

```python
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    # Application
    app_name: str = "My API"
    environment: str = "development"
    debug: bool = False
    version: str = "1.0.0"

    # Server
    host: str = "0.0.0.0"
    port: int = 8000
    workers: int = 4

    # Database
    database_url: str
    db_pool_size: int = 20
    db_max_overflow: int = 10

    # Redis
    redis_url: str
    cache_ttl: int = 300

    # RabbitMQ
    rabbitmq_url: str

    # Security
    jwt_secret: str
    jwt_algorithm: str = "HS256"
    access_token_expire_minutes: int = 30

    # External Services
    user_service_url: str = ""
    payment_service_url: str = ""

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"

@lru_cache
def get_settings() -> Settings:
    return Settings()

# Environment-specific configs
class DevelopmentSettings(Settings):
    debug: bool = True
    log_level: str = "DEBUG"

class ProductionSettings(Settings):
    debug: bool = False
    log_level: str = "INFO"
    workers: int = 8

def get_settings_for_environment() -> Settings:
    env = os.getenv("ENVIRONMENT", "development")
    if env == "production":
        return ProductionSettings()
    return DevelopmentSettings()
```

---

## Production Checklist

### Pre-Deployment

- [ ] All tests passing (unit, integration, E2E)
- [ ] Security scan completed (dependencies, SAST)
- [ ] Database migrations tested
- [ ] Environment variables configured
- [ ] Secrets rotated and secured
- [ ] Resource limits set (CPU, memory)
- [ ] Health checks configured

### Monitoring

- [ ] Structured logging enabled
- [ ] Metrics exposed (/metrics endpoint)
- [ ] Distributed tracing configured
- [ ] Alerting rules defined
- [ ] Dashboards created

### Performance

- [ ] Connection pooling configured (DB, Redis)
- [ ] Caching strategy implemented
- [ ] Query optimization verified (no N+1)
- [ ] Rate limiting enabled
- [ ] Compression enabled (gzip)

### Security

- [ ] HTTPS enforced
- [ ] CORS configured correctly
- [ ] Input validation on all endpoints
- [ ] Authentication required where needed
- [ ] Secrets not in code/logs
- [ ] Security headers set (CSP, HSTS, etc.)

### Reliability

- [ ] Circuit breakers configured
- [ ] Retry logic with backoff
- [ ] Graceful shutdown handling
- [ ] Database connection recovery
- [ ] Queue consumer recovery

### Observability

```python
# Graceful shutdown
import signal

@app.on_event("startup")
async def startup():
    # Initialize connections
    await database.connect()
    await redis.connect()
    await rabbitmq.connect()

    # Register signal handlers
    for sig in (signal.SIGTERM, signal.SIGINT):
        signal.signal(sig, lambda s, f: asyncio.create_task(shutdown()))

async def shutdown():
    logger.info("Shutting down gracefully...")

    # Stop accepting new requests
    # Wait for in-flight requests (handled by uvicorn)

    # Close connections
    await rabbitmq.close()
    await redis.close()
    await database.disconnect()

    logger.info("Shutdown complete")
```
