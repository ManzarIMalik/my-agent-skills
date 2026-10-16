# Microservices Patterns

Pseudocode conventions are in [SKILL.md](../SKILL.md#pseudocode-conventions). Resilience libraries per language: [language-adaptation.md](language-adaptation.md#ecosystem-toolbox).

## Table of Contents

1. [Service Boundaries](#service-boundaries)
2. [Service-to-Service Communication](#service-to-service-communication)
3. [Timeouts and Deadlines](#timeouts-and-deadlines)
4. [Retry with Exponential Backoff](#retry-with-exponential-backoff)
5. [Circuit Breaker Pattern](#circuit-breaker-pattern)
6. [Bulkheads, Fallbacks, Load Shedding](#bulkheads-fallbacks-load-shedding)
7. [Service Discovery and Load Balancing](#service-discovery-and-load-balancing)
8. [API Gateway and BFF](#api-gateway-and-bff)
9. [Data Ownership](#data-ownership)

---

## Service Boundaries

- **One service per bounded context**, owned by one team, with its own data store. If two services must always deploy together or share tables, they're one service.
- **Prefer coarse-grained services**. Chains of synchronous calls (A → B → C → D) multiply latency and failure probability.
- **Design for failure**: every remote call can be slow, fail, or succeed without you knowing (timeout after the server committed).

---

## Service-to-Service Communication

| Style | Use for | Notes |
|---|---|---|
| **Synchronous HTTP/REST** | Queries needing an immediate answer, public-ish internal APIs | Simple, debuggable; couples availability |
| **gRPC** | High-volume internal calls, streaming, strict typed contracts across languages | Codegen in every major language from one `.proto` |
| **Asynchronous events/commands** | State changes others react to, long-running work | Decouples availability; eventual consistency — see [event-driven.md](event-driven.md) |

### Client adapter (HTTP)

Wrap each remote service in a client adapter that implements a port, so use cases don't know about HTTP:

```
interface UserDirectory {                       // port, defined by the consuming service
  getUser(id: UserId) -> UserSummary?
}

class HttpUserDirectory(http: HttpClient, baseUrl: string) implements UserDirectory {
  // http client created once and reused (connection pooling), with default timeouts
  getUser(id) -> UserSummary? {
    response = http.get(baseUrl + "/v1/users/" + id,
                        headers = { "User-Agent": "order-service/1.4.2", traceparent: currentTrace() },
                        timeout = 2 seconds)
    match response.status {
      200 -> return UserSummary.fromJson(response.body)       // map to *this* service's model
      404 -> return none
      else -> fail ExternalServiceError("user-service", response.status, response.bodySnippet)
    }
  }
}
```

### gRPC contract

```protobuf
// contracts/proto/user/v1/user_service.proto
syntax = "proto3";
package user.v1;

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}

message GetUserRequest  { string user_id = 1; }
message GetUserResponse { string id = 1; string email = 2; string name = 3; string tier = 4; }
```

Generate the client in the consuming service's language (`protoc`/`buf` plugins exist for Go, Java/Kotlin, C#, TS, Python, Rust, PHP, Ruby, Swift, Dart, Elixir) and wrap it in an adapter exactly like the HTTP one. Use TLS (mTLS inside a mesh) — never plaintext channels in production.

---

## Timeouts and Deadlines

The single most important resilience setting. **Every** network call (HTTP, gRPC, DB, cache, broker) needs a timeout.

- Set **connect** and **overall request** timeouts explicitly; library defaults are often infinite or very long.
- **Propagate deadlines**: if the incoming request has 800 ms left, downstream calls get less than that, not a fresh 30 s. (gRPC deadlines, Go `context.WithTimeout`, .NET `CancellationToken`, `AbortSignal`, Kotlin `withTimeout`.)
- Timeouts should be shorter the deeper the call is in the chain.

---

## Retry with Exponential Backoff

Retry only **transient** failures (timeouts, connection resets, 502/503/504, 429 with `Retry-After`) and only **idempotent** operations (or ones made idempotent with keys).

```
retry(operation, maxAttempts = 3, baseDelay = 200ms, maxDelay = 5s) {
  for attempt in 1..maxAttempts {
    try {
      return operation()
    } catch error {
      if not isTransient(error) or attempt == maxAttempts: rethrow error
      delay = random(0, min(maxDelay, baseDelay × 2^(attempt-1)))      // "full jitter"
      if error.retryAfter is present: delay = max(delay, error.retryAfter)
      log.warn("retrying", attempt, delay, error)
      sleep(delay)                                                   // non-blocking sleep in async runtimes
    }
  }
}
```

- **Jitter** prevents synchronised retry storms.
- **Retry at one layer only** — retries at every hop multiply (3 × 3 × 3 = 27 attempts).
- Use a **retry budget** (e.g. retries ≤ 10% of requests) to avoid amplifying an outage.
- Prefer the ecosystem's library (Resilience4j, Polly, cockatiel, tenacity, `backoff`, …) over hand-rolled loops.

---

## Circuit Breaker Pattern

Stops calling a dependency that's clearly failing, giving it room to recover and failing fast for callers.

```
CLOSED ──(failure rate ≥ threshold over window)──► OPEN ──(cool-down elapsed)──► HALF_OPEN
   ▲                                                                                │
   └────────────(trial calls succeed)──────────────────────────────(trial fails)───┘──► OPEN
```

```
class CircuitBreaker(failureThreshold = 5, coolDown = 30 seconds, halfOpenMaxCalls = 1, clock) {
  state = CLOSED; failures = 0; openedAt = none; halfOpenInFlight = 0     // guard with a lock/atomics if shared

  call(operation) {
    if state == OPEN:
      if clock.now() - openedAt < coolDown: fail CircuitOpen()
      state = HALF_OPEN
    if state == HALF_OPEN and halfOpenInFlight >= halfOpenMaxCalls: fail CircuitOpen()

    isTrial = (state == HALF_OPEN)                   // remember locally: state may change before `finally`
    if isTrial: halfOpenInFlight += 1
    try {
      result = operation()
      onSuccess(); return result
    } catch error {
      if countsAsFailure(error): onFailure()        // don't count 4xx client errors
      rethrow error
    } finally {
      if isTrial: halfOpenInFlight -= 1             // otherwise the counter leaks and the breaker never closes
    }
  }

  onSuccess() { failures = 0; state = CLOSED }
  onFailure() {
    failures += 1
    if state == HALF_OPEN or failures >= failureThreshold:
      state = OPEN; openedAt = clock.now()
      log.warn("circuit opened"); metrics.gauge("circuit_state", OPEN)
  }
}

// Usage: decorate the client adapter
class ResilientUserDirectory(inner: UserDirectory, breaker: CircuitBreaker, cache: Cache) implements UserDirectory {
  getUser(id) {
    try { return breaker.call(() -> retry(() -> inner.getUser(id))) }
    catch CircuitOpen { return cachedOrDegraded(id) }            // explicit fallback
  }
}
```

Production breakers usually use a **sliding window failure rate** (e.g. ≥ 50% of the last 20 calls) rather than a raw count, and expose state as a metric.

---

## Bulkheads, Fallbacks, Load Shedding

- **Bulkhead**: separate connection pools / concurrency limits per dependency, so a slow dependency can't exhaust all threads/connections.
- **Fallback**: a deliberate degraded response — cached value, default, partial result, "try again later". Make fallbacks visible in metrics.
- **Load shedding**: reject early (429/503) when queues or concurrency exceed limits, rather than accepting work you'll time out on.
- **Rate limiting** outbound calls to respect partners' quotas.

---

## Service Discovery and Load Balancing

Usually provided by the platform — don't hand-roll a registry:

| Platform | Mechanism |
|---|---|
| Kubernetes | `Service` DNS names (`http://user-service.default.svc`) + kube-proxy or a service mesh |
| Service mesh (Istio, Linkerd, Consul Connect) | Sidecar/ambient proxies: discovery, mTLS, retries, timeouts, circuit breaking, traffic shifting |
| Cloud | Load balancers, Cloud Map / Service Connect, internal DNS |
| VMs / bare metal | Consul, etcd, or DNS SRV with health checks |

If retries/timeouts/breakers are configured in a mesh, don't also configure aggressive ones in code — pick one layer and document it.

---

## API Gateway and BFF

- **API gateway**: single entry point for external clients — TLS termination, authentication, rate limiting, routing, request/response transformation (Kong, Envoy, NGINX, Traefik, cloud gateways).
- **Backend for Frontend (BFF)**: a thin service per client type (web, mobile) that aggregates and shapes backend data for that client. Keeps chatty composition off the device and lets each client evolve independently. Keep business logic out of it.

---

## Data Ownership

- **Each service owns its data; no other service reads its tables.** Sharing a database couples deploys and schemas.
- To use another service's data: **call its API**, or **subscribe to its events** and keep a local read-only projection of the fields you need.
- Cross-service consistency is **eventual** — design UX and business processes for it (pending states, sagas — see [event-driven.md](event-driven.md#saga-pattern)).
- Reporting across services goes through a data pipeline / warehouse, not cross-service joins.
