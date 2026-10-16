# Project Structure

## Table of Contents

1. [Guiding Ideas](#guiding-ideas)
2. [Layered Modular Monolith](#layered-modular-monolith)
3. [Language-Specific Layout Notes](#language-specific-layout-notes)
4. [Microservices Structure](#microservices-structure)
5. [When to Use Each](#when-to-use-each)
6. [Developer Environment](#developer-environment)

---

## Guiding Ideas

- **Organise by business capability first, technical layer second.** `orders/domain`, `orders/application` scale better than one giant `domain/` folder shared by every feature. Each module (bounded context) owns its data and exposes a narrow public API to other modules.
- **Make boundaries enforceable**, not just conventional: separate packages/modules/projects/crates with explicit visibility, plus an architecture-test or lint rule where the ecosystem supports it (ArchUnit, NetArchTest, dependency-cruiser, `import-linter`, Go `internal/`, Rust crate privacy, Deptrac for PHP, Packwerk for Rails, Elixir `boundary`).
- **Follow the ecosystem's conventional layout** — newcomers should find things where they expect them.

---

## Layered Modular Monolith

A language-neutral template. Rename folders to the ecosystem's conventions (see below).

```
project/
├── src/                                   # or app/, lib/, internal/, cmd/ + pkg/, Sources/ …
│   ├── <module-a>/                        # one bounded context, e.g. orders
│   │   ├── domain/                        # entities, value objects, domain events, domain services, domain errors
│   │   ├── application/                   # use cases, commands/queries, DTOs, ports (interfaces)
│   │   ├── infrastructure/                # adapters: repositories, ORM mappings, cache, messaging, external API clients
│   │   └── interface/                     # HTTP/gRPC/GraphQL handlers, CLI commands, message consumers, request/response schemas
│   ├── <module-b>/
│   ├── shared/                            # truly shared kernel only: IDs, Money, clock, base errors (keep it small)
│   └── bootstrap/                         # composition root: config loading, DI wiring, server startup
├── migrations/                            # versioned schema migrations
├── tests/                                 # or co-located tests, per ecosystem convention
│   ├── unit/
│   ├── integration/
│   ├── contract/
│   ├── e2e/
│   └── performance/
├── deploy/                                # Dockerfile, compose, Kubernetes/Helm, Terraform/Pulumi
├── docs/
│   ├── adr/                               # architecture decision records
│   ├── api/                               # OpenAPI / protobuf / GraphQL schema
│   └── runbooks/
├── scripts/                               # dev/CI helper scripts
├── <build manifest + lockfile>            # package.json+lock, go.mod+go.sum, Cargo.toml+Cargo.lock, pom.xml, …
├── .env.example                           # documented config, never real secrets
└── README.md                              # how to run, test, and deploy
```

**Dependency rule inside each module:** `interface → application → domain`, and `infrastructure → application/domain` (implementing ports). Nothing depends on `interface` or `infrastructure` except `bootstrap`.

**Cross-module rule:** module A calls module B only through B's public application API (or B's events) — never B's repositories or tables.

---

## Language-Specific Layout Notes

| Ecosystem | Conventional shape | Notes |
|---|---|---|
| **Go** | `cmd/<app>/main.go`, `internal/<module>/…` | Package by feature (`internal/order`, `internal/billing`); keep layers as files or sub-packages only when they grow. `internal/` enforces privacy. |
| **Rust** | Cargo workspace: `crates/domain`, `crates/app`, `crates/infra`, `crates/api`, binary crate wires them | Crate boundaries enforce the dependency rule at compile time. |
| **Java / Kotlin** | Maven/Gradle multi-module or one module with packages `com.acme.orders.{domain,application,infrastructure,web}` | Spring Modulith or ArchUnit to enforce boundaries. |
| **C# / .NET** | Solution with projects `Orders.Domain`, `Orders.Application`, `Orders.Infrastructure`, `Orders.Api` | Project references enforce direction; `internal` + `InternalsVisibleTo` for tests. |
| **TypeScript / Node** | `src/modules/<module>/{domain,application,infrastructure,http}` or a monorepo (pnpm/Nx/Turborepo) with packages | Use path aliases and dependency-cruiser/eslint boundaries rules. |
| **Python** | `src/<package>/<module>/{domain,application,infrastructure,api}` with `pyproject.toml` | `import-linter` contracts enforce layering. |
| **PHP** | Laravel/Symfony default under `app/` or `src/`, with `src/<Module>/{Domain,Application,Infrastructure,Http}` for complex domains | Deptrac for layer rules. |
| **Ruby / Rails** | Rails defaults (`app/models`, `app/controllers`) plus `app/services` or `app/domain`; engines or Packwerk packs for modules | Don't fight Rails for simple apps. |
| **Elixir / Phoenix** | Contexts in `lib/my_app/<context>.ex` (public API) with internals in `lib/my_app/<context>/`; web layer in `lib/my_app_web/` | Contexts *are* the module boundaries. |
| **Swift / Kotlin / Dart apps** | Feature modules (SPM packages, Gradle modules, Dart packages) | See [client-applications.md](client-applications.md). |

---

## Microservices Structure

```
services/
├── user-service/
│   ├── src/                       # same internal layering as the monolith module
│   ├── tests/
│   ├── Dockerfile
│   └── <build manifest>
├── order-service/
├── payment-service/
├── notification-service/
├── contracts/                     # schemas shared *by contract*, not by code
│   ├── events/                    # JSON Schema / Avro / Protobuf event definitions
│   ├── proto/                     # gRPC service definitions
│   └── openapi/                   # HTTP API specs
├── gateway/                       # API gateway / BFF config (Kong, Envoy, Traefik, cloud gateway, or custom)
└── compose.yaml                   # local stack
```

Services can be written in different languages; that's why shared things live as **schemas** that each service generates code from, not as a shared code library. A shared library is acceptable for genuinely cross-cutting, slow-changing concerns (logging setup, tracing middleware) within one language — but treat changes to it as breaking changes for every consumer.

---

## When to Use Each

| Approach | Best For | Trade-offs |
|---|---|---|
| **Modular monolith** | Most new products, small-to-medium teams, domains still being discovered | One deployable, simple ops, easy refactoring across modules. Needs discipline to keep module boundaries clean. |
| **Microservices** | Many teams needing independent deploys, parts with very different scaling or reliability needs, established domain boundaries | Independent scaling and ownership. Pays for it with network failures, distributed data, eventual consistency, and much heavier operations. |
| **Serverless functions** | Spiky/event-driven workloads, glue code, low-traffic APIs | Minimal ops, pay-per-use. Cold starts, vendor coupling, harder local testing. The same layering applies inside each function. |

**Start with a modular monolith; extract a service when a module has a clear, independent reason to be separate.** A clean module boundary makes later extraction cheap; a distributed monolith is the worst of both worlds.

---

## Developer Environment

Whatever the language, a new contributor should be able to go from clone to running tests with one or two documented commands.

- **Pin the toolchain**: `.tool-versions` (asdf/mise), `.nvmrc`, `rust-toolchain.toml`, `go.mod`'s `go`/`toolchain` directive, `global.json`, `.python-version`, Gradle/Maven wrapper.
- **Commit the lockfile** for applications.
- **Use the ecosystem's standard tooling**: e.g. npm/pnpm, Go modules, Cargo, Maven/Gradle, `dotnet`, uv/Poetry, Composer, Bundler, Mix, SwiftPM.
- **Provide a task runner** (`Makefile`, `justfile`, `package.json` scripts, `mise` tasks, Gradle tasks) with consistent verbs: `setup`, `dev`, `test`, `lint`, `fmt`, `migrate`.
- **Run dependencies locally with containers** (`compose.yaml`) or a dev container so everyone has the same database/cache/broker versions.
- **Formatter + linter in CI**, configured once (Prettier/ESLint/Biome, gofmt/golangci-lint, rustfmt/clippy, ktlint/detekt/Spotless, `dotnet format`, Ruff, PHP-CS-Fixer/PHPStan, RuboCop, `mix format`/Credo, SwiftLint).

```
# Example setup script (shell) — adapt the commands to the stack
set -e
<install toolchain from pinned version file>
<install dependencies from lockfile>
docker compose up -d db cache
<run migrations>
<run test suite>
echo "Development environment ready"
```
