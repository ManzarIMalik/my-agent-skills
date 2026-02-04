# Project Structure

## Table of Contents

1. [Monolithic Application](#monolithic-application)
2. [Microservices Structure](#microservices-structure)
3. [When to Use Each](#when-to-use-each)

---

## Monolithic Application

```
project/
├── src/
│   └── <domain>/
│       ├── __init__.py
│       ├── domain/                    # Core business logic
│       │   ├── entities/              # Business entities
│       │   ├── value_objects/         # Immutable domain values
│       │   ├── events/                # Domain events
│       │   ├── services/              # Domain services
│       │   └── exceptions/            # Domain-specific exceptions
│       ├── application/               # Use cases & orchestration
│       │   ├── use_cases/             # Application use cases
│       │   ├── interfaces/            # Abstract repositories, ports
│       │   ├── dtos/                  # Data transfer objects
│       │   └── services/              # Application services
│       ├── infrastructure/            # External implementations
│       │   ├── database/
│       │   │   ├── models/            # ORM models
│       │   │   ├── repositories/      # Concrete repository implementations
│       │   │   └── migrations/        # Alembic migrations
│       │   ├── cache/                 # Redis, in-memory cache
│       │   ├── messaging/             # RabbitMQ, Kafka adapters
│       │   ├── external_apis/         # Third-party API clients
│       │   └── storage/               # S3, file storage
│       └── presentation/              # External interfaces
│           ├── api/
│           │   ├── v1/                # Versioned API routes
│           │   ├── v2/
│           │   ├── dependencies.py    # DI factory functions
│           │   └── middleware/        # Custom middleware
│           ├── grpc/                  # gRPC services
│           ├── cli/                   # CLI commands
│           └── schemas/               # Pydantic request/response models
├── tests/
│   ├── unit/                          # Fast, isolated tests
│   ├── integration/                   # Tests with real dependencies
│   ├── e2e/                           # End-to-end API tests
│   ├── performance/                   # Load tests
│   ├── fixtures/                      # Shared test fixtures
│   └── conftest.py
├── deployment/
│   ├── docker/
│   │   ├── Dockerfile
│   │   └── docker-compose.yml
│   ├── kubernetes/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   └── terraform/                     # Infrastructure as code
├── docs/
│   ├── api/                           # OpenAPI specs
│   ├── architecture/                  # Architecture decision records
│   └── runbooks/                      # Operational guides
├── scripts/
│   ├── seed_data.py
│   ├── migrate.py
│   └── performance_test.py
├── pyproject.toml
├── poetry.lock
├── .env.example
└── alembic.ini
```

---

## Microservices Structure

```
services/
├── user-service/
│   ├── src/
│   ├── tests/
│   ├── Dockerfile
│   └── pyproject.toml
├── order-service/
├── payment-service/
├── notification-service/
├── shared/                            # Shared libraries
│   ├── common/
│   │   ├── events/                    # Event schemas
│   │   ├── exceptions/
│   │   └── utils/
│   └── proto/                         # gRPC proto files
├── api-gateway/                       # Kong, Traefik, or custom
└── docker-compose.yml
```

---

## When to Use Each

| Approach          | Best For                                                | Trade-offs                                                           |
| ----------------- | ------------------------------------------------------- | -------------------------------------------------------------------- |
| **Monolith**      | Early-stage projects, small teams, simple domains       | Easier to develop, deploy, debug. Can become unwieldy at scale.      |
| **Microservices** | Large teams, complex domains, independent scaling needs | Better scalability and team autonomy. Higher operational complexity. |

**Start monolithic, extract services when clearly beneficial.**

---

## Development Environment Setup

### Using uv (Recommended)

[uv](https://docs.astral.sh/uv/) is an extremely fast Python package manager written in Rust.

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create virtual environment
uv venv .venv

# Activate virtual environment
source .venv/bin/activate  # Linux/macOS
# .venv\Scripts\activate   # Windows

# Install dependencies from pyproject.toml
uv pip install -e ".[dev]"

# Or install from requirements.txt
uv pip install -r requirements.txt

# Add a package
uv pip install fastapi

# Sync dependencies (install exact versions from lock file)
uv pip sync requirements.txt
```

### Project Configuration with uv

```toml
# pyproject.toml
[project]
name = "my-project"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.109.0",
    "uvicorn>=0.27.0",
    "sqlalchemy>=2.0.0",
    "pydantic>=2.5.0",
    "redis>=5.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=4.1.0",
    "ruff>=0.1.0",
    "mypy>=1.8.0",
]

[tool.uv]
dev-dependencies = [
    "pytest>=7.4.0",
    "ruff>=0.1.0",
]
```

### Quick Start Script

```bash
#!/bin/bash
# scripts/setup.sh

set -e

# Check if uv is installed
if ! command -v uv &> /dev/null; then
    echo "Installing uv..."
    curl -LsSf https://astral.sh/uv/install.sh | sh
fi

# Create and activate venv
uv venv .venv
source .venv/bin/activate

# Install dependencies
uv pip install -e ".[dev]"

# Run migrations
alembic upgrade head

echo "✅ Development environment ready!"
```
