# Production Deployment

Containers and Kubernetes are used as the reference platform because they're language-neutral; the principles carry over to PaaS (Fly.io, Render, Heroku, Cloud Run, App Service, Elastic Beanstalk), serverless, and VMs.

## Table of Contents

1. [Build Principles](#build-principles)
2. [Container Images](#container-images)
3. [Local Stack with Compose](#local-stack-with-compose)
4. [Kubernetes Deployment](#kubernetes-deployment)
5. [Configuration Management](#configuration-management)
6. [Graceful Shutdown](#graceful-shutdown)
7. [Release Strategies](#release-strategies)
8. [Production Checklist](#production-checklist)

---

## Build Principles

- **Build once, deploy everywhere**: one immutable artifact (image) promoted through environments; only configuration differs ([twelve-factor](https://12factor.net/)).
- **Reproducible builds**: pinned toolchain, lockfile, pinned base image digests, CI-only builds.
- **CI pipeline**: format/lint → unit tests → build → integration/contract tests → security scans → image publish (tagged with git SHA) → deploy to staging → smoke tests → promote to production.

---

## Container Images

### Multi-stage Dockerfile (template)

```dockerfile
# ---- Build stage: full toolchain ----
FROM <language-sdk-image>:<pinned-version> AS build
WORKDIR /src

# 1. Copy only dependency manifests first so this layer is cached between code changes
COPY <manifest> <lockfile> ./
RUN <install/download dependencies>

# 2. Copy source and build
COPY . .
RUN <build / compile / bundle for production>

# ---- Runtime stage: minimal image ----
FROM <minimal-runtime-image>:<pinned-version> AS runtime
WORKDIR /app

# Run as a non-root user
RUN <create non-root user "app">          # many minimal/distroless images ship a nonroot user already
COPY --from=build --chown=app:app <build output> ./
USER app

EXPOSE 8080
# Exec form so the process is PID 1 and receives SIGTERM directly
ENTRYPOINT ["<start command>"]
```

| Stack | Build image | Runtime image | Build output |
|---|---|---|---|
| **Go** | `golang` | `gcr.io/distroless/static` or `scratch` (with `CGO_ENABLED=0`) | single static binary |
| **Rust** | `rust` | `distroless/cc` or `debian-slim` (`musl` target for `scratch`) | single binary |
| **Java / Kotlin** | `eclipse-temurin:<v>-jdk` (+ Maven/Gradle) | `eclipse-temurin:<v>-jre` or distroless Java; or GraalVM native image | fat JAR / layered JAR / native binary |
| **C# / .NET** | `mcr.microsoft.com/dotnet/sdk` | `mcr.microsoft.com/dotnet/aspnet` (or `runtime-deps` for self-contained / chiseled images) | `dotnet publish` output |
| **Node / TypeScript** | `node:<lts>` | `node:<lts>-slim` or distroless Node | compiled JS + production `node_modules` only |
| **Python** | `python:<v>` (+ uv/Poetry) | `python:<v>-slim` | virtualenv copied from build stage + source |
| **PHP** | `composer` + `php:<v>-cli` | `php:<v>-fpm` (+ nginx) or FrankenPHP | `vendor/` (no dev deps) + source, OPcache enabled |
| **Ruby** | `ruby:<v>` | `ruby:<v>-slim` | `BUNDLE_WITHOUT="development:test" bundle install` + precompiled assets |
| **Elixir** | `hexpm/elixir` | `debian-slim` / `alpine` | `mix release` |

Image hygiene:
- `.dockerignore` excludes `.git`, local env files, build caches, tests, `node_modules`, etc.
- No secrets in images or build args; use BuildKit secret mounts if the build needs credentials.
- Scan images (Trivy, Grype) in CI; rebuild regularly to pick up base-image patches.
- Let the platform do health checks (Kubernetes probes) — a Dockerfile `HEALTHCHECK` is only useful for plain Docker/Compose, and needs a binary (curl/wget) that minimal images may not contain.

---

## Local Stack with Compose

```yaml
# compose.yaml
services:
  api:
    build: { context: ., target: runtime }
    ports: ["8080:8080"]
    environment:
      DATABASE_URL: postgres://app:app@postgres:5432/app
      REDIS_URL: redis://redis:6379/0
      BROKER_URL: amqp://app:app@rabbitmq:5672/
    depends_on:
      postgres: { condition: service_healthy }
      redis:    { condition: service_healthy }

  postgres:
    image: postgres:16
    environment: { POSTGRES_USER: app, POSTGRES_PASSWORD: app, POSTGRES_DB: app }
    volumes: [postgres_data:/var/lib/postgresql/data]
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U app -d app"], interval: 5s, timeout: 5s, retries: 5 }

  redis:
    image: redis:7-alpine
    healthcheck: { test: ["CMD", "redis-cli", "ping"], interval: 5s, timeout: 5s, retries: 5 }

  rabbitmq:
    image: rabbitmq:3-management
    environment: { RABBITMQ_DEFAULT_USER: app, RABBITMQ_DEFAULT_PASS: app }

volumes:
  postgres_data:
```

Credentials here are for local development only.

---

## Kubernetes Deployment

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels: { app: api }
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxUnavailable: 0, maxSurge: 1 }
  selector:
    matchLabels: { app: api }
  template:
    metadata:
      labels: { app: api }
    spec:
      securityContext: { runAsNonRoot: true, seccompProfile: { type: RuntimeDefault } }
      containers:
        - name: api
          image: registry.example.com/api:3f2c9ab          # immutable tag (git SHA) or digest
          ports: [{ containerPort: 8080 }]
          envFrom:
            - configMapRef: { name: api-config }
            - secretRef:    { name: api-secrets }
          resources:
            requests: { cpu: "250m", memory: "256Mi" }
            limits:   { memory: "512Mi" }                   # memory limit; CPU limits often cause throttling
          securityContext: { allowPrivilegeEscalation: false, readOnlyRootFilesystem: true, capabilities: { drop: ["ALL"] } }   # mount an emptyDir at /tmp if the app writes temp files
          startupProbe:   { httpGet: { path: /health/live,  port: 8080 }, periodSeconds: 5, failureThreshold: 30 }
          livenessProbe:  { httpGet: { path: /health/live,  port: 8080 }, periodSeconds: 10 }
          readinessProbe: { httpGet: { path: /health/ready, port: 8080 }, periodSeconds: 5 }
          lifecycle:
            preStop: { sleep: { seconds: 5 } }              # let endpoints update before SIGTERM handling (k8s ≥ 1.30; else exec sleep)
      terminationGracePeriodSeconds: 30
```

Runtime tuning to match container limits: JVM `-XX:MaxRAMPercentage`, Node `--max-old-space-size`, Go `GOMEMLIMIT` (and `GOMAXPROCS`, which is container-aware by default from Go 1.25), .NET server GC settings, worker/process counts for Python/Ruby/PHP servers sized to CPU requests.

### Service, Config, Secrets

```yaml
apiVersion: v1
kind: Service
metadata: { name: api }
spec:
  selector: { app: api }
  ports: [{ port: 80, targetPort: 8080 }]
---
apiVersion: v1
kind: ConfigMap
metadata: { name: api-config }
data:
  LOG_LEVEL: "info"
  ENVIRONMENT: "production"
# Secrets: create from a secret manager (External Secrets Operator, Sealed Secrets, CSI driver) —
# never commit real secret values to manifests.
```

### Autoscaling and Availability

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: api }
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: api }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: api } }
```

Scale queue consumers on queue depth/lag (KEDA) rather than CPU. Package manifests with Helm or Kustomize and deploy via GitOps (Argo CD, Flux) or your CD pipeline. Run migrations as a separate Job/pipeline step before rolling out new pods, following expand/contract (see [database-patterns.md](database-patterns.md#migrations)).

---

## Configuration Management

Configuration comes from the **environment** (env vars, mounted files, secret manager), is **validated at startup**, and is exposed to the app as one **typed object** injected where needed — never read ad hoc from `env` throughout the codebase.

```
record Settings {
  environment:        "development" | "staging" | "production"
  port:               int            = 8080
  logLevel:           string         = "info"
  databaseUrl:        Secret<string>                 // required, never logged
  dbPoolSize:         int            = 10
  redisUrl:           string
  cacheTtl:           Duration       = 5 minutes
  brokerUrl:          Secret<string>
  oidcIssuer:         Url
  apiAudience:        string
  userServiceUrl:     Url
  requestTimeout:     Duration       = 2 seconds
}

loadSettings() -> Settings {
  settings = parse(Settings, from = [defaults, configFile(optional), environmentVariables])
  validate(settings)               // fail fast with a clear message listing every missing/invalid key
  return settings
}
```

Language tools: Spring `@ConfigurationProperties`, .NET Options pattern (`IOptions<T>` + validation), Zod/`envalid` over `process.env`, `pydantic-settings`, Go `envconfig`/`koanf`/`viper`, Rust `config`/`figment`/`envy`, Laravel `config/` + `.env`, Rails credentials + ENV, Elixir `runtime.exs`.

- Same code path in every environment; differences are config values, not `if env == "prod"` branches scattered in code.
- Feature flags (OpenFeature-compatible providers, LaunchDarkly, Unleash, Flagsmith) for runtime toggles, not config redeploys.

---

## Graceful Shutdown

On `SIGTERM` (or the platform's equivalent):

```
onSignal(SIGTERM, SIGINT) {
  log.info("shutdown started")
  readiness.markNotReady()                // load balancer stops sending new traffic
  sleep(preStopDelay)                     // optional: allow endpoint propagation
  httpServer.stopAccepting()
  httpServer.awaitInFlight(timeout = 20 seconds)
  consumers.stopFetching(); consumers.awaitInFlight(timeout = 20 seconds)   // finish & ack current messages
  backgroundJobs.drain()
  outboxRelay.stop()
  broker.close(); cache.close(); dbPool.close()
  telemetry.flush()                       // don't lose the last spans/metrics/logs
  log.info("shutdown complete")
  exit(0)
}
```

- The total must fit inside the platform's grace period (`terminationGracePeriodSeconds`).
- Ensure the app process is PID 1 or uses a minimal init (`tini`, `dumb-init`) so signals arrive.
- Most frameworks have hooks for this (Spring graceful shutdown, `IHostApplicationLifetime`, Node `server.close()` + signal handlers, Uvicorn/Gunicorn graceful timeout, Go `http.Server.Shutdown(ctx)`, Axum `with_graceful_shutdown`, Puma/Sidekiq quiet+stop, OTP application stop callbacks).

---

## Release Strategies

| Strategy | How | Good for |
|---|---|---|
| **Rolling** | Replace instances gradually | Default; requires backward-compatible changes |
| **Blue/green** | Two full environments; switch traffic | Fast rollback; costs double capacity briefly |
| **Canary** | Small % of traffic to the new version, watch metrics, then ramp | High-risk changes; needs good observability (Argo Rollouts, Flagger, service mesh) |
| **Feature flags** | Deploy dark, release by toggling | Decoupling deploy from release; gradual rollout per user segment |

Always have a **tested rollback path** — and remember database migrations must be compatible with the previous version for rollback to work.

---

## Production Checklist

### Pre-Deployment
- [ ] All tests passing (unit, integration, contract, E2E smoke)
- [ ] Dependency, image, and SAST scans clean or triaged
- [ ] Migrations tested against production-like data; backward compatible
- [ ] Configuration validated; secrets in a secret manager and rotated
- [ ] Resource requests/limits set; runtime tuned to them
- [ ] Health probes (startup, liveness, readiness) configured

### Monitoring
- [ ] Structured JSON logs with trace IDs
- [ ] RED metrics + business metrics exported
- [ ] Distributed tracing with context propagation (including through queues)
- [ ] SLOs, burn-rate alerts, runbooks
- [ ] Dashboards with deploy markers

### Performance
- [ ] Connection pools sized (DB, cache, HTTP clients)
- [ ] Caching where measured to help
- [ ] No N+1 queries on hot paths; slow-query logging on
- [ ] Rate limiting and request size limits
- [ ] Response compression; static assets on a CDN

### Security
- [ ] HTTPS/TLS everywhere; HSTS
- [ ] CORS allow-list
- [ ] Input validation on every endpoint; output encoding
- [ ] Authentication and object-level authorization enforced
- [ ] No secrets or PII in code, images, or logs
- [ ] Security headers set (CSP, HSTS, nosniff, …)

### Reliability
- [ ] Timeouts on every outbound call; retries with backoff + jitter (one layer only)
- [ ] Circuit breakers / bulkheads for critical dependencies
- [ ] Graceful shutdown within the grace period
- [ ] Idempotent message consumers; DLQ with replay tooling
- [ ] Backups automated **and restore-tested**; disaster recovery plan documented
