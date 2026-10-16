# Code Style & Standards

Language-neutral standards. Where a convention differs by language (naming, file layout, formatting), the ecosystem's convention wins — see [language-adaptation.md](language-adaptation.md#concept--idiom-map).

## Table of Contents

1. [General Principles](#general-principles)
2. [Naming](#naming)
3. [Module and Import Organization](#module-and-import-organization)
4. [Error Handling](#error-handling)
5. [Comments and Documentation](#comments-and-documentation)

---

## General Principles

1. **Guard clauses / early return** — handle invalid cases first so the main path is un-nested.
2. **Explicit types at public boundaries** — compiler-checked where possible; annotations/type hints in dynamic languages.
3. **Non-blocking I/O where the runtime expects it** — see [language-adaptation.md](language-adaptation.md#concurrency-and-async).
4. **Explicit over implicit** — no hidden globals, monkey-patching, or reflection-driven magic in business code.
5. **Small, focused functions** — if a function needs a comment to explain its sections, those sections probably want to be functions. Aim for something that fits on a screen; there's no magic line count.
6. **Immutability by default** — mutate only where it's clearer or measurably faster.
7. **Single responsibility** — one reason to change.
8. **Automate style** — formatter and linter in CI, so reviews discuss design rather than whitespace.

```
// ❌ Nested
process(order) {
  if order is present {
    if order.items not empty {
      if order.status == PENDING {
        ...main logic...
      }
    }
  }
}

// ✅ Guard clauses
process(order) {
  if order is none:            fail InvalidInput("order required")
  if order.items is empty:     fail RuleViolated("order_must_have_items", "Order has no items")
  if order.status != PENDING:  fail RuleViolated("order_must_be_pending", "Order is " + order.status)
  ...main logic...
}
```

---

## Naming

Casing follows the language (see the idiom map). These rules apply everywhere:

- **Name by domain meaning, not by type or mechanism**: `overdueInvoices`, not `invoiceList2`; `PriceCalculator`, not `PriceHelper`.
- **Functions are verbs, types are nouns, booleans read as predicates**: `calculateTotal`, `Order`, `isActive` / `hasShipped`.
- **Constants describe intent and unit**: `MAX_RETRY_ATTEMPTS`, `REQUEST_TIMEOUT_SECONDS` (or a typed `Duration`, which is better still).
- **Avoid junk-drawer modules**: `utils`, `helpers`, `common`, `misc`, `manager`, `processor`. Prefer `email_sender`, `price_calculator`, `order_validator`.
- **Use the ubiquitous language** — the same words the business uses, in code, APIs, and database. If the business says "policy holder", don't call it `customer` in code.
- **Type aliases / newtypes for IDs** (`UserId`, `OrderId`) prevent passing one ID where another is expected — use a real distinct type where the language supports it (newtype in Rust/Haskell, value class in Kotlin, branded type in TS, struct wrapper in Go/C#).

---

## Module and Import Organization

Group imports in this order (and let the formatter enforce it):

1. Standard library
2. Third-party packages
3. Your own application modules

Additional rules:
- **Import direction must respect layers** — a domain file importing an ORM or HTTP package is a design smell.
- **Avoid circular dependencies** between modules; they usually reveal a missing abstraction or a misplaced responsibility.
- **Prefer explicit imports** over wildcard imports.
- **Expose a module's public API deliberately** (index/barrel file, `mod.rs` re-exports, package-level exports, context module) and keep internals private.

---

## Error Handling

### Error Taxonomy

```
// Domain errors — expected business outcomes
error DomainError
  NotFound(entityType, entityId)              -> "Order 123 not found"
  RuleViolated(rule, reason)                  -> "Business rule 'sufficient_inventory' violated: ..."
    InsufficientInventory(productId, requested, available)
  Conflict(reason)                            -> e.g. duplicate email, version mismatch

// Infrastructure errors — external system failures
error InfrastructureError
  DatabaseUnavailable(cause)
  CacheUnavailable(cause)
  ExternalServiceError(service, statusCode, message)
  Timeout(operation, after)
```

Express this as an exception hierarchy, an enum/sum type, or sentinel + wrapped errors depending on the language — see [language-adaptation.md](language-adaptation.md#errors).

### Handling Rules

```
getUser(id) -> UserDTO {
  user = users.getById(id)                      // may fail with DatabaseUnavailable
  if user is none: fail NotFound("User", id)
  return UserDTO.from(user)
}

// The edge (middleware / global error handler) maps categories to transport responses:
on NotFound             -> 404
on RuleViolated         -> 422
on Conflict             -> 409
on DatabaseUnavailable  -> log.error(..., cause); 503
on anything else        -> log.error(..., stack trace); 500
```

1. **Never swallow errors silently.** An empty catch / ignored error return / `_ = err` hides failures. If you deliberately ignore one, write down why.
2. **Catch or match narrowly.** Handle only the error types you can do something about; let the rest propagate to the edge.
3. **Add context when propagating** (which operation, which IDs) — without leaking secrets or personal data.
4. **Translate at boundaries.** Infrastructure adapters convert driver-specific errors into your `InfrastructureError` types; handlers convert domain errors into transport responses. Domain code never mentions HTTP status codes.
5. **Fail fast on programmer errors** (broken invariants, impossible states) rather than limping on with corrupt data.
6. **Don't use errors for control flow** in hot paths when a normal return value expresses the outcome better.

---

## Comments and Documentation

- Comment **why**, not **what** — the code shows what.
- Document public APIs with the ecosystem's doc format (Javadoc/KDoc, XML docs, TSDoc/JSDoc, docstrings, GoDoc, rustdoc, PHPDoc, YARD, ExDoc, DocC, dartdoc).
- Record significant design decisions as ADRs in `docs/adr/` (context, decision, consequences).
- Keep a README that answers: what is this, how do I run it, how do I test it, how is it deployed.
