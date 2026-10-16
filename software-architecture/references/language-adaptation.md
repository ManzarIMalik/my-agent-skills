# Language Adaptation

How to turn the pseudocode in this skill into idiomatic code in a specific language. The goal is to preserve the *architectural intent* (boundaries, dependency direction, invariants, explicit errors) while writing code that looks native to its ecosystem.

## Table of Contents

1. [Adaptation Workflow](#adaptation-workflow)
2. [Concept → Idiom Map](#concept--idiom-map)
3. [Errors](#errors)
4. [Concurrency and Async](#concurrency-and-async)
5. [Dependency Injection Styles](#dependency-injection-styles)
6. [Ecosystem Toolbox](#ecosystem-toolbox)
7. [Paradigm Notes](#paradigm-notes)
8. [Common Translation Mistakes](#common-translation-mistakes)

---

## Adaptation Workflow

1. **Detect the stack.** Read the manifest and a few existing files. Note language version, framework, ORM/data layer, test framework, linter/formatter config, and folder conventions.
2. **Follow what exists.** If the project already has a pattern for errors, DI, or layout, extend it rather than introducing a second one. Consistency within a codebase matters more than matching this skill.
3. **Map each concept** using the table below. If a concept has no natural equivalent (e.g. classes in Go, mutation in Elixir), express the *intent* differently instead of emulating the missing feature.
4. **Choose libraries from the ecosystem**, preferring the framework's built-ins and the project's existing dependencies before adding new ones.
5. **Keep the domain framework-free** in every language: no ORM annotations, HTTP types, or framework base classes in domain types unless the team has consciously accepted that trade-off (common and reasonable in Rails/Django/Laravel-style apps — say so explicitly when you take it).

---

## Concept → Idiom Map

| Concept | Java / Kotlin | C# | TypeScript | Python | Go | Rust | PHP | Ruby | Swift | Dart | Elixir |
|---|---|---|---|---|---|---|---|---|---|---|---|
| **Port / interface** | `interface` | `interface` | `interface` / type | `Protocol` / `ABC` | small `interface`, declared by the consumer | `trait` | `interface` | duck typing (document the contract; test it) | `protocol` | `abstract class` / `abstract interface class` | `@behaviour` + callbacks |
| **Immutable value / record** | `record` / `data class` (`val`) | `record` / `readonly struct` | `readonly` props, `as const`, `Object.freeze` | `@dataclass(frozen=True)` | unexported fields + constructor, value receivers | struct without `mut`, derive `Clone, PartialEq, Eq` | `readonly class` (8.2+) | `Data.define` / frozen `Struct` | `struct` with `let` | `final` fields, `@immutable` | structs are immutable by default |
| **Optional value (`T?`)** | `Optional<T>` / `T?` | `T?` (nullable refs on) | `T \| undefined` | `T \| None` | `(T, bool)` or pointer, or error | `Option<T>` | `?T` | `nil` | `T?` | `T?` | `nil` or `{:ok, v}`/`:error` |
| **Sum type / closed set** | `sealed interface` / `sealed class` | `abstract record` hierarchy, enums | discriminated union | `Enum`, `Literal`, union types | const + type, or interface with marker method | `enum` | `enum` (8.1+) | symbols / classes | `enum` with associated values | `sealed class` | tagged tuples, atoms |
| **Generic repository** | generics | generics | generics | `Generic[T]` / PEP 695 | generics (1.18+), but prefer concrete per-aggregate interfaces | generics + traits | templates via PHPDoc (`@template`) | not needed | generics + associated types | generics | not needed |
| **Scoped resource / transaction** | try-with-resources, `@Transactional` | `using`, `TransactionScope` | `try/finally`, `await using` (TS 5.2+), callback helper | `with` / `async with` | `defer tx.Rollback()` then `tx.Commit()` | RAII / `Drop`, closure helper | `try/finally`, closure helper | block (`transaction do … end`) | `defer` | `try/finally` | `Repo.transaction(fn -> … end)` / `Ecto.Multi` |
| **Visibility for "internal" details** | package-private / `internal` | `internal` | non-exported module members | leading `_` (convention) | lowercase identifiers | `pub(crate)` / private | `private` / `protected` | `private` | `internal` / `fileprivate` | leading `_` (library-private) | `defp`, `@moduledoc false` |
| **Naming (types / functions / constants)** | `PascalCase` / `camelCase` / `UPPER_SNAKE` | `PascalCase` / `PascalCase` / `PascalCase` | `PascalCase` / `camelCase` / `UPPER_SNAKE` or `camelCase` | `PascalCase` / `snake_case` / `UPPER_SNAKE` | `PascalCase` (exported) / `camelCase` / `PascalCase` or `camelCase` | `PascalCase` / `snake_case` / `UPPER_SNAKE` | `PascalCase` / `camelCase` / `UPPER_SNAKE` | `PascalCase` / `snake_case` / `UPPER_SNAKE` | `PascalCase` / `camelCase` / `camelCase` | `PascalCase` / `camelCase` / `camelCase` | `PascalCase` modules / `snake_case` / module attrs |

For languages not listed (Scala, F#, Clojure, Haskell, OCaml, Zig, C++, Lua, …) apply the same reasoning: find the language's way to express an abstraction boundary, an immutable value, an absent value, a closed set of cases, and a scoped resource.

---

## Errors

The architecture needs three things from errors, regardless of mechanism: **typed categories** (so the edge can map them), **context** (what failed, with which IDs), and **no silent swallowing**.

| Style | Languages | How to express this skill's error categories |
|---|---|---|
| **Exceptions** | Java, Kotlin, C#, Python, TypeScript/JS, PHP, Ruby, Swift (`throws`), Dart | A small hierarchy: `DomainError` → `NotFound`, `RuleViolated`, `Conflict`; `InfrastructureError` → `Unavailable`, `Timeout`. Catch only what you can handle; translate in a global handler/middleware/exception filter. Never use a bare catch-all that discards the error. |
| **Result / error values** | Rust (`Result<T, E>`), Go (`(T, error)`), Elixir (`{:ok, v} / {:error, r}`), Kotlin/TS/Swift/C# when the team uses `Result`/`Either` libraries | Define an error enum/type per layer. Wrap with context as it propagates (`fmt.Errorf("get user %s: %w", id, err)` in Go; `thiserror`/`anyhow` with `.context()` in Rust; `with` chains in Elixir). Use `errors.Is/As` (Go) or `match` on variants to map at the edge. |
| **Let it crash + supervision** | Elixir/Erlang | Return tagged tuples for *expected* failures (not found, invalid); let *unexpected* failures crash the process and rely on supervisors to restart. |

Rule of thumb: **expected business outcomes** (not found, insufficient stock) are part of the function's contract — model them explicitly (typed exception or error variant). **Unexpected faults** (null deref, corrupted state) should surface loudly.

---

## Concurrency and Async

The pseudocode writes I/O as plain calls. Translate according to the runtime:

| Runtime model | Languages | Guidance |
|---|---|---|
| `async/await` on an event loop | TypeScript/Node, Python (asyncio), C#, Rust (Tokio), Swift, Dart | Make every I/O path async end-to-end; never call blocking I/O on the loop (offload CPU-heavy or blocking calls to a worker pool). Propagate cancellation (`AbortSignal`, `CancellationToken`, task cancellation). |
| Coroutines | Kotlin | `suspend` functions, structured concurrency with scopes; use `Dispatchers.IO` for blocking calls. |
| Lightweight threads | Go (goroutines), Java 21+ (virtual threads), Elixir/Erlang (processes) | Write straightforward blocking-style code. Pass `context.Context` (Go) for deadlines/cancellation; use bounded concurrency (worker pools, semaphores, `errgroup`). |
| Thread pools / sync | Classic Java/Spring MVC, PHP-FPM, Ruby (Puma), Django sync | Blocking calls are fine; size connection pools to the worker count; push slow work to background jobs. |

Whatever the model: **every remote call needs a timeout**, and concurrent fan-out needs a **bound**.

---

## Dependency Injection Styles

DI is a principle (receive collaborators, don't construct them), not a framework. Choose the lightest option that fits:

| Style | When | Examples |
|---|---|---|
| **Manual constructor injection + composition root** | Default for most services, all languages | Go `main.go` wiring, Rust `AppState`, a TS `container.ts`, Python `bootstrap.py` |
| **Framework container** | When the framework is built around it | Spring, ASP.NET Core `IServiceCollection`, NestJS modules, Angular, Laravel container, Symfony, Quarkus/Micronaut, Dagger/Hilt (Android), FastAPI `Depends` |
| **Function parameters / closures** | Functional or small codebases | Pass functions or records of functions; partial application |
| **Context / environment objects** | Request-scoped values | Go `context.Context` (for deadlines and request values, *not* for core dependencies), Elixir config + behaviours, Reader pattern |

Avoid global singletons and service locators: they hide dependencies and make tests order-dependent.

---

## Ecosystem Toolbox

Common, well-maintained choices — always prefer what the project already uses. Verify current versions against the ecosystem's docs before pinning.

| Concern | Java / Kotlin (JVM) | C# / .NET | TypeScript / Node | Python | Go | Rust | PHP | Ruby | Elixir |
|---|---|---|---|---|---|---|---|---|---|
| Web / API | Spring Boot, Quarkus, Micronaut, Ktor | ASP.NET Core (Minimal APIs / MVC) | NestJS, Fastify, Express, Hono | FastAPI, Django, Flask, Litestar | `net/http`, chi, Echo, Gin | Axum, Actix Web | Laravel, Symfony | Rails, Sinatra, Hanami | Phoenix |
| Data access | JPA/Hibernate, jOOQ, Exposed, Spring Data JDBC | EF Core, Dapper | Prisma, Drizzle, TypeORM, Kysely | SQLAlchemy, Django ORM | `database/sql`, pgx, sqlc, GORM | SQLx, Diesel, SeaORM | Eloquent, Doctrine | Active Record, Sequel, ROM | Ecto |
| Migrations | Flyway, Liquibase | EF Core migrations | Prisma Migrate, Drizzle Kit, Knex | Alembic, Django migrations | golang-migrate, goose, Atlas | sqlx-cli, Diesel CLI, refinery | Laravel migrations, Doctrine Migrations | Rails migrations | Ecto migrations |
| Validation | Jakarta Bean Validation | FluentValidation, DataAnnotations | Zod, Valibot, class-validator | Pydantic | go-playground/validator | `validator`, `garde`, serde | Laravel validation, Symfony Validator | Active Model validations, dry-validation | Ecto changesets |
| Unit tests | JUnit 5, Kotest, Mockito, MockK | xUnit, NUnit, Moq, NSubstitute | Vitest, Jest, node:test | pytest | `testing`, testify | built-in `#[test]`, mockall | PHPUnit, Pest | RSpec, Minitest | ExUnit, Mox |
| Integration (real deps) | Testcontainers | Testcontainers | Testcontainers | Testcontainers | Testcontainers | testcontainers-rs | Testcontainers | Testcontainers | Testcontainers / sandboxed `Ecto.Adapters.SQL.Sandbox` |
| Logging | SLF4J + Logback/Log4j2 (JSON encoder) | `ILogger` + Serilog | pino, winston | structlog, stdlib `logging` | `log/slog`, zap, zerolog | `tracing` | Monolog | Rails logger, semantic_logger | `Logger` |
| Metrics / tracing | Micrometer, OpenTelemetry | OpenTelemetry .NET | OpenTelemetry JS | OpenTelemetry Python, prometheus_client | OpenTelemetry Go, prometheus client | OpenTelemetry Rust, `metrics` | OpenTelemetry PHP | OpenTelemetry Ruby | OpenTelemetry Erlang/Elixir, Telemetry |
| Resilience | Resilience4j | Polly / Microsoft.Extensions.Resilience | cockatiel, p-retry | tenacity, pybreaker | `cenkalti/backoff`, `sony/gobreaker` | `backoff`, `tower` middleware | Laravel retry helpers, Guzzle middleware | `retriable`, circuitbox | `:fuse`, built-in supervision |
| Background jobs | Spring Batch, Quartz, JobRunr | Hangfire, Quartz.NET, MassTransit | BullMQ | Celery, RQ, Dramatiq, arq | River, Asynq | apalis, custom Tokio tasks | Laravel Queues, Symfony Messenger | Sidekiq, Solid Queue, GoodJob | Oban |

Infrastructure components in this skill (PostgreSQL, Redis, Kafka, RabbitMQ, Docker, Kubernetes, Prometheus, OpenTelemetry) are language-neutral — they're examples of a *role* (relational store, cache, broker, container runtime, …) and can be swapped for equivalents.

---

## Paradigm Notes

**Functional languages and styles (Elixir, F#, Haskell, Clojure, Scala, functional TS):** the layering still holds, but it becomes "functional core, imperative shell". Entities become immutable data plus pure functions that return a new state *and* the events produced (`confirm(order) -> (order', [OrderConfirmed])`). Repositories become modules/behaviours or functions passed in. Units of work become composed transactional operations (`Ecto.Multi`, a `ConnectionIO`/`Kleisli` program).

**Go:** prefer small interfaces defined where they're *used* (by the consumer), concrete return types, and packages organised by domain capability (`order/`, `billing/`) rather than by technical layer. Don't build class hierarchies or generic base repositories; a few concrete repository interfaces are clearer. Use `internal/` to enforce boundaries.

**Rust:** ownership makes aggregates natural — methods that take `&mut self` enforce single-writer invariants. Model states as enums or typestate (`Order<Pending>` → `Order<Confirmed>`) so invalid transitions don't compile. Put ports behind traits; use generics for zero-cost static dispatch or `dyn Trait`/`Arc<dyn Trait>` where you need runtime swapping.

**Convention-over-configuration frameworks (Rails, Django, Laravel, Phoenix):** the framework's model layer *is* the persistence layer. For simple domains, use it directly and keep business logic in models/service objects. When complexity grows, introduce plain domain objects and use-case/service classes *alongside* the framework, not instead of it. Don't fight the framework to achieve textbook purity on a CRUD app.

**Dynamic languages (Python, Ruby, JS, PHP):** interfaces are contracts enforced by tests (and optionally type checkers: mypy/pyright, TypeScript, Sorbet/RBS, PHPStan/Psalm). Turn on the strictest checker the team will tolerate at public boundaries.

---

## Common Translation Mistakes

- **Java-in-Go / Java-in-Python:** deep inheritance, abstract base classes for everything, `IUserRepository` naming in languages that don't use the `I` prefix, getters/setters everywhere.
- **Emulating exceptions in Result languages** (or vice versa): using `panic`/`unwrap` for expected errors in Go/Rust, or returning error codes in Java/C#.
- **Mixing async models:** blocking calls inside an event loop, or bolting `async` onto a language/runtime where blocking is idiomatic (Go, virtual-thread Java).
- **Over-abstracting small projects:** a generic repository + unit of work + mediator on top of an ORM that already provides those, for a three-table CRUD app.
- **Ignoring the framework's conventions** for project layout, configuration, and testing, which makes the project harder for newcomers in that ecosystem.
- **Copying a library choice from another ecosystem** instead of using the native equivalent from the toolbox above.
