# API Design

Transport-level design that applies to any server framework. Pseudocode conventions are in [SKILL.md](../SKILL.md#pseudocode-conventions); framework choices per language are in [language-adaptation.md](language-adaptation.md#ecosystem-toolbox).

## Table of Contents

1. [Choosing a Style](#choosing-a-style)
2. [Resource and Endpoint Design](#resource-and-endpoint-design)
3. [Handler Structure](#handler-structure)
4. [Request/Response Schemas](#requestresponse-schemas)
5. [Errors](#errors)
6. [Pagination, Filtering, Sorting](#pagination-filtering-sorting)
7. [Idempotency and Concurrency](#idempotency-and-concurrency)
8. [Authentication in Handlers](#authentication-in-handlers)
9. [API Versioning and Evolution](#api-versioning-and-evolution)
10. [Contract-First and Documentation](#contract-first-and-documentation)

---

## Choosing a Style

| Style | Good for | Watch out for |
|---|---|---|
| **REST/HTTP + JSON** | Public APIs, broad client compatibility, cacheable reads | Chatty clients, over/under-fetching |
| **gRPC (Protobuf)** | Internal service-to-service, streaming, strict contracts, polyglot codegen | Browser support needs a proxy (gRPC-Web/Connect); harder to debug by hand |
| **GraphQL** | Many client shapes over the same data (mobile + web), aggregating backends | Query cost control, N+1 in resolvers (use dataloaders), HTTP caching is harder |
| **Async messaging / webhooks** | Notifications, long-running work, decoupled integrations | Delivery guarantees, ordering, retries — see [event-driven.md](event-driven.md) |

---

## Resource and Endpoint Design

```
POST   /v1/users                 create                 → 201 + Location header
GET    /v1/users/{id}            fetch one              → 200 | 404
GET    /v1/users?status=active   list / filter          → 200 (paginated)
PATCH  /v1/users/{id}            partial update         → 200 | 404 | 409 | 422
PUT    /v1/users/{id}            full replace           → 200 | 201
DELETE /v1/users/{id}            delete                 → 204 | 404
POST   /v1/orders/{id}/confirm   domain action          → 200 | 409 | 422
GET    /v1/users/{id}/orders     sub-collection         → 200
```

- **Nouns for resources, HTTP methods for verbs.** For domain actions that aren't CRUD (confirm, cancel, refund), a sub-resource action `POST /orders/{id}/cancel` is clearer than overloading `PATCH`.
- **Plural, kebab- or lower-case paths**; consistent JSON field casing across the whole API (pick camelCase or snake_case once).
- **Use status codes precisely**: 400 malformed, 401 not authenticated, 403 not allowed, 404 not found, 409 conflict/state, 422 validation/rule failure, 429 rate limited, 503 dependency down.
- **Safe and idempotent methods** (`GET`, `PUT`, `DELETE`) must behave so; make `POST` idempotent with keys when retries are likely.
- **Timestamps in ISO-8601 UTC**, money as decimal strings or integer minor units plus currency — never floats.

---

## Handler Structure

A handler does four things: **parse/validate input → call one use case → map the result → map errors**. No business logic, no SQL.

```
// presentation/http/user_routes
routes(createUser: CreateUserUseCase, getUser: GetUserUseCase, listUsers: ListUsersUseCase,
       updateUser: UpdateUserUseCase, auth: Authenticator) {

  POST "/v1/users" (req) {
    body = validate(req.body, UserCreateRequest)              // 422 on failure (handled centrally)
    user = createUser.execute(body.toCommand())               // EmailTaken -> 409 (handled centrally)
    return respond(201, UserResponse.from(user), headers = { Location: "/v1/users/" + user.id })
  }

  GET "/v1/users/{id}" (req) {
    return respond(200, UserResponse.from(getUser.execute(parseUserId(req.path.id))))
  }

  GET "/v1/users" (req) {
    page = PageRequest.parse(req.query, defaultSize = 20, maxSize = 100)
    result = listUsers.execute(page)
    return respond(200, PageResponse.from(result, UserResponse.from))
  }

  PATCH "/v1/users/{id}" (req) {
    principal = auth.require(req)                                   // 401
    id = parseUserId(req.path.id)
    if principal.userId != id and not principal.has("users:write"): // 403
      fail Forbidden("Cannot update other users")
    user = updateUser.execute(id, validate(req.body, UserUpdateRequest).toCommand())
    return respond(200, UserResponse.from(user))
  }
}
```

**Where it lives per framework**: controllers (Spring, ASP.NET, NestJS, Laravel, Rails, Phoenix), route handlers (Express/Fastify/Hono, FastAPI, Flask), `http.HandlerFunc` (Go), handler functions with extractors (Axum/Actix). Use the framework's validation and error-mapping hooks rather than hand-rolling them in each handler.

---

## Request/Response Schemas

Keep transport schemas **separate from domain entities** so the API contract can evolve independently of internals and you never accidentally expose fields like `passwordHash`.

```
record UserCreateRequest {
  email:    string   (required, format email, max 254)
  name:     string   (required, 1..100 chars)
  password: string   (required, 12..128 chars)   // length beats composition rules; check against breached-password lists
  toCommand() -> CreateUserCommand
}

record UserUpdateRequest {     // PATCH: every field optional
  name:  string?  (1..100 chars)
  email: string?  (format email)
}

record UserResponse {
  id: string, email: string, name: string, tier: string, isActive: bool, createdAt: string (ISO-8601 UTC)
  static from(dto: UserDTO) -> UserResponse
}

record PageResponse<T> {
  items: List<T>
  page: { size: int, nextCursor: string? }        // cursor style
  // or { number: int, size: int, totalItems: int, totalPages: int }  // offset style
}
```

Validation layers:
1. **Shape validation at the edge** (types, required, lengths, formats) via the framework's validator or a schema library (Bean Validation, FluentValidation, Zod, Pydantic, go-playground/validator, serde + validator, Laravel/Symfony validators, Ecto changesets).
2. **Business validation in the domain** (value object constructors, aggregate methods).

---

## Errors

Use one consistent error envelope across the API. [RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457) is a good default:

```
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{
  "type": "https://api.example.com/problems/validation",
  "title": "Validation failed",
  "status": 422,
  "detail": "One or more fields are invalid.",
  "instance": "/v1/users",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "errors": [ { "field": "email", "message": "must be a valid email" } ]
}
```

Map domain/infrastructure errors to this format in **one central place** (middleware, exception filter, error handler). Never return stack traces or internal messages to clients; log them with the trace ID instead.

---

## Pagination, Filtering, Sorting

| Approach | Pros | Cons |
|---|---|---|
| **Cursor / keyset** (`?after=<opaque cursor>&limit=20`) | Stable under inserts, fast on large tables (uses an index) | No "jump to page 37" |
| **Offset** (`?page=3&size=20`) | Simple, supports page numbers | Slow for deep pages, rows shift on insert |

- Always enforce a **maximum page size**.
- Make cursors **opaque** (encode the sort key + ID).
- Whitelist filter and sort fields; never pass them raw into a query.
- `GET /v1/orders?status=pending&sort=-createdAt&limit=50`

---

## Idempotency and Concurrency

**Idempotency keys** make retries of non-idempotent operations safe (payments, order creation):

```
POST /v1/payments
Idempotency-Key: 9f1c6c1e-…

handler(req) {
  key = req.headers["Idempotency-Key"] ?? fail BadRequest("Idempotency-Key required")
  scope = req.principal.id

  // Claim the key atomically (unique constraint / SET NX) so two concurrent retries can't both execute
  claimed = idempotencyStore.claim(key, scope, requestHash = hash(req.body), ttl = 24 hours)
  if not claimed:
    existing = idempotencyStore.get(key, scope)
    if existing.requestHash != hash(req.body): fail Conflict("Key reused with a different request")
    if existing.status == IN_PROGRESS:         fail Conflict("Request with this key is still processing")   // client retries later
    return existing.response                                                                             // replay

  try {
    response = processPayment.execute(...)
    idempotencyStore.complete(key, scope, response)   // ideally in the same transaction as the effect
    return response
  } catch error {
    idempotencyStore.release(key, scope)             // let a retry run again after a failure
    rethrow error
  }
}
```

**Optimistic concurrency** prevents lost updates: return an `ETag` (or `version`) on reads, require `If-Match` on writes, respond `412 Precondition Failed` (or `409`) on mismatch.

---

## Authentication in Handlers

Authenticate in middleware/filters, expose the **principal** to handlers, and authorise close to the operation:

```
middleware authenticate(req, next) {
  token = bearerToken(req.headers.Authorization) ?? return respond(401)
  claims = tokenVerifier.verify(token)   // signature, issuer, audience, expiry; fails -> 401
  req.principal = Principal(userId = claims.sub, roles = claims.roles, scopes = claims.scope)
  return next(req)
}

requireScope(scope) = middleware (req, next) {
  if scope not in req.principal.scopes: return respond(403)
  return next(req)
}
```

Object-level checks ("can this user see *this* order?") belong in the use case or a policy object, not only in route-level middleware — see [security.md](security.md#authorization).

---

## API Versioning and Evolution

- **Prefer additive, backward-compatible change**: add optional fields and new endpoints; never rename or remove fields, change types, or tighten validation in place.
- **Clients must ignore unknown fields** (tolerant reader).
- **Version when you must break**: URL path (`/v2/…`) is the most visible and cache-friendly; header or media-type versioning (`Accept: application/vnd.example.v2+json`) keeps URLs stable.
- **Deprecate deliberately**: announce, add `Deprecation` and `Sunset` headers (RFC 9745 / RFC 8594) with a link to migration docs, monitor usage, then remove.
- **gRPC/Protobuf**: never reuse or renumber field tags; mark removed fields `reserved`.
- **GraphQL**: evolve continuously with `@deprecated(reason: …)` instead of versions.

---

## Contract-First and Documentation

- Describe the API in a machine-readable contract — **OpenAPI** for HTTP, **.proto** for gRPC, **SDL** for GraphQL, **AsyncAPI** for events — and keep it in the repo.
- Either generate server stubs/clients from the contract, or generate the contract from code annotations; either way, **check it in CI** (lint it, diff it for breaking changes, run contract tests).
- Publish interactive docs (Swagger UI, Redoc, Scalar, GraphiQL) and examples for every endpoint.
