# Observability

Pseudocode conventions are in [SKILL.md](../SKILL.md#pseudocode-conventions). **OpenTelemetry** is the vendor-neutral standard for traces, metrics, and logs with SDKs for every major language; Prometheus-style metrics and JSON logs are understood by every backend. Per-language libraries: [language-adaptation.md](language-adaptation.md#ecosystem-toolbox).

## Table of Contents

1. [Goals](#goals)
2. [Structured Logging](#structured-logging)
3. [Metrics](#metrics)
4. [Distributed Tracing](#distributed-tracing)
5. [Health Checks](#health-checks)
6. [Alerting and SLOs](#alerting-and-slos)

---

## Goals

You should be able to answer, from telemetry alone: *Is it working? For whom is it broken? Where is the time going? What changed?*

- **Logs**: discrete events with context — what happened.
- **Metrics**: cheap aggregated numbers over time — how much, how fast, how often.
- **Traces**: the path of one request across components — where.

Correlate all three with a shared **trace ID**.

---

## Structured Logging

Emit **one JSON object per line** to stdout; let the platform ship it. Never build log lines with string concatenation of business data.

```json
{"ts":"2026-03-14T09:26:53.589Z","level":"info","msg":"order placed","service":"order-service","version":"1.4.2","env":"prod","trace_id":"4bf92f3577b34da6a3ce929d0e0e4736","span_id":"00f067aa0ba902b7","request_id":"c0a8012e","order_id":"ord_123","customer_id":"cus_456","total":"59.90","currency":"EUR","duration_ms":42}
```

```
// Bind context once; every subsequent log line carries it
placeOrder(cmd) {
  log = logger.with(orderId = cmd.orderId, customerId = cmd.customerId)
  log.info("placing order")
  try {
    result = doPlaceOrder(cmd)
    log.info("order placed", total = result.total)
  } catch error {
    log.error("order placement failed", error = error)   // include error type + stack
    rethrow error
  }
}

// Request-context middleware: request ID + trace ID propagated through async boundaries
middleware requestContext(req, next) {
  requestId = req.headers["X-Request-ID"] ?? newId()
  withLogContext(requestId = requestId, traceId = currentTraceId(), method = req.method, route = req.routeTemplate) {
    start = monotonicNow()
    response = next(req)
    logger.info("request completed", status = response.status, durationMs = monotonicNow() - start)
    response.headers["X-Request-ID"] = requestId
    return response
  }
}
```

Context propagation mechanism per language: MDC (Java/SLF4J), `ILogger` scopes / `Activity` (.NET), `AsyncLocalStorage` (Node), `contextvars` (Python), `context.Context` + `slog` (Go), `tracing` spans (Rust), `Logger.metadata` (Elixir), Monolog processors (PHP), tagged logging (Rails).

Rules:
- **Levels**: `error` = needs attention; `warn` = unexpected but handled; `info` = significant business/lifecycle events; `debug` = developer detail (off in prod by default).
- **Log the route template** (`/users/{id}`), not the raw path, to keep cardinality sane.
- **No secrets or unnecessary PII** in logs; redact at the logger level as a backstop.
- Don't log-and-rethrow at every layer — log once where the error is handled.

---

## Metrics

Start with the **RED** method per service/endpoint (Rate, Errors, Duration) and **USE** for resources (Utilization, Saturation, Errors).

```
// Instruments (names follow OpenTelemetry semantic conventions where they exist)
requests     = meter.counter("http.server.requests", labels = [method, route, status_class])
latency      = meter.histogram("http.server.request.duration", unit = "s", labels = [method, route],
                               buckets = [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10])
inFlight     = meter.upDownCounter("http.server.active_requests")
dbDuration   = meter.histogram("db.client.operation.duration", unit = "s", labels = [operation, table])
cacheResults = meter.counter("cache.requests", labels = [cache, result])   // result = hit | miss | error
ordersPlaced = meter.counter("orders.placed", labels = [channel])           // business metric

middleware metrics(req, next) {
  inFlight.add(1); start = monotonicNow()
  try {
    response = next(req)
    return response
  } finally {
    labels = { method: req.method, route: req.routeTemplate, status_class: statusClass(response) }  // "2xx", "5xx"; no response (exception) counts as "5xx"
    requests.add(1, labels)
    latency.record(seconds(monotonicNow() - start), labels without status_class)
    inFlight.add(-1)
  }
}
```

- **Expose** via an OTLP exporter or a Prometheus `/metrics` endpoint (on an internal port).
- **Cardinality discipline**: never use user IDs, order IDs, raw URLs, or error messages as label values.
- Use **histograms** for latency (so you can compute p95/p99), not averages.
- Add **business metrics** (orders placed, payments failed, sign-ups) — they often reveal incidents before technical metrics do.
- Most frameworks have auto-instrumentation for HTTP, DB, and cache metrics; use it before writing your own.

---

## Distributed Tracing

```
// Setup at startup (composition root)
setupTelemetry(serviceName, serviceVersion, environment, otlpEndpoint) {
  resource = { "service.name": serviceName, "service.version": serviceVersion, "deployment.environment": environment }
  tracerProvider = TracerProvider(resource, exporter = OtlpExporter(otlpEndpoint), sampler = ParentBased(TraceIdRatio(0.1)))
  enableAutoInstrumentation([httpServer, httpClient, database, cache, messaging])
}

// Manual span around meaningful business work
capturePayment(orderId, amount) {
  span tracer.startActiveSpan("payment.capture") {
    span.setAttribute("order.id", orderId)
    span.setAttribute("payment.amount", amount.toString())
    try {
      result = gateway.capture(amount)
      span.setAttribute("payment.id", result.paymentId)
      return result
    } catch error {
      span.recordException(error)
      span.setStatus(ERROR, error.message)
      rethrow error
    }
  }
}
```

- **Propagate context** across every hop using W3C Trace Context headers (`traceparent`, `tracestate`) — HTTP, gRPC metadata, and **message headers** for async events (so traces continue through queues).
- Auto-instrumentation handles HTTP/DB/clients in most stacks (OpenTelemetry Java agent, .NET instrumentation packages, `@opentelemetry/auto-instrumentations-node`, `opentelemetry-instrument` for Python, eBPF or contrib packages for Go, `tracing-opentelemetry` for Rust).
- **Sample** intelligently: head-based ratio sampling for volume, tail-based sampling in the collector to keep all errors and slow traces.
- Run an **OpenTelemetry Collector** between apps and backends to batch, sample, redact, and route without app changes.

---

## Health Checks

Separate the questions the platform asks:

| Endpoint | Question | Checks | On failure |
|---|---|---|---|
| `/health/live` (liveness) | Is the process alive and not deadlocked? | Nothing external — just "can I respond" | Platform **restarts** the instance |
| `/health/ready` (readiness) | Can I serve traffic right now? | Critical dependencies (DB reachable, warmed up, not shutting down) | Removed from load balancer, **not** restarted |
| `/health/startup` (optional) | Have I finished starting? | Migrations/caches loaded | Liveness checks wait |
| `/health` (detailed, internal) | Status of each component | All dependencies, with latency | For humans/dashboards |

```
enum Health { HEALTHY, DEGRADED, UNHEALTHY }
record ComponentHealth { name, status: Health, latencyMs: number?, message: string? }

class HealthChecker(checks: List<(name, critical: bool, check: () -> void)>, timeoutPerCheck = 2 seconds) {
  report() -> { status: Health, components: List<ComponentHealth>, version, uptimeSeconds } {
    components = runConcurrently(checks, each -> {
      start = monotonicNow()
      try { withTimeout(timeoutPerCheck) { each.check() }
            return ComponentHealth(each.name, HEALTHY, latencyMs = monotonicNow() - start) }
      catch error {
            return ComponentHealth(each.name, each.critical ? UNHEALTHY : DEGRADED, message = error.type) }
    })
    overall = any UNHEALTHY ? UNHEALTHY : any DEGRADED ? DEGRADED : HEALTHY
    return { overall, components, version = build.version, uptimeSeconds = uptime() }
  }
}

checker = HealthChecker([
  ("database", critical = true,  () -> db.execute("SELECT 1")),
  ("cache",    critical = false, () -> cache.ping()),      // app works without cache, just slower
])

GET /health/live  -> 200 { "status": "ok" }
GET /health/ready -> shuttingDown or checker.report().status == UNHEALTHY ? 503 : 200
GET /health       -> checker.report()  (internal only; don't expose dependency details publicly)
```

- **Liveness must not check dependencies** — a database outage would otherwise restart every instance in a loop.
- Cache health results for a few seconds so probes don't hammer dependencies.
- Flip readiness to failing at the **start of graceful shutdown** so traffic drains before exit (see [deployment.md](deployment.md#graceful-shutdown)).

---

## Alerting and SLOs

- Define **SLOs** on user-visible behaviour: e.g. "99.9% of `POST /orders` succeed within 500 ms over 30 days".
- **Alert on symptoms** (error-budget burn rate, high error rate, high latency) rather than causes (CPU 80%). Cause-based signals belong on dashboards.
- Every alert should be **actionable** and link to a **runbook** in `docs/runbooks/`.
- Dashboards per service: RED metrics, saturation (pools, queues, memory), dependency health, deploy markers, key business metrics.
