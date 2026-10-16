# Database Patterns

Data-layer guidance that holds across ORMs, query builders, and raw SQL. Examples use standard SQL (PostgreSQL dialect where it matters) because SQL is the common language across ecosystems. Library choices per language: [language-adaptation.md](language-adaptation.md#ecosystem-toolbox).

## Table of Contents

1. [Choosing a Store](#choosing-a-store)
2. [Schema Design](#schema-design)
3. [Mapping Between Domain and Storage](#mapping-between-domain-and-storage)
4. [Connections and Transactions](#connections-and-transactions)
5. [Query Optimization](#query-optimization)
6. [Concurrency Control](#concurrency-control)
7. [Migrations](#migrations)

---

## Choosing a Store

| Need | Default choice |
|---|---|
| Transactions, relationships, ad-hoc queries | **Relational** (PostgreSQL, MySQL/MariaDB, SQL Server, SQLite for embedded/edge) |
| Flexible documents, per-document atomicity | Document (MongoDB, Couchbase, or JSON columns in Postgres) |
| Very high write throughput, known access patterns | Wide-column / key-value (Cassandra, ScyllaDB, DynamoDB) |
| Full-text and faceted search | Search engine (OpenSearch/Elasticsearch, Meilisearch, Typesense, Postgres FTS for modest needs) |
| Caching, ephemeral data, rate limits | In-memory store (Redis, Valkey, Memcached) |
| Time series / analytics | TimescaleDB, ClickHouse, a warehouse |

Start with one relational database unless a requirement clearly rules it out. Each extra store adds consistency, backup, and operational burden.

---

## Schema Design

```sql
CREATE TABLE users (
  id            UUID PRIMARY KEY,
  email         VARCHAR(254) NOT NULL,
  name          VARCHAR(100) NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  tier          VARCHAR(16)  NOT NULL DEFAULT 'bronze' CHECK (tier IN ('bronze','silver','gold')),
  is_active     BOOLEAN      NOT NULL DEFAULT TRUE,
  created_at    TIMESTAMPTZ  NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ  NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_users_email ON users (lower(email));

CREATE TABLE orders (
  id           UUID PRIMARY KEY,
  customer_id  UUID NOT NULL REFERENCES users(id),
  status       VARCHAR(16) NOT NULL CHECK (status IN ('pending','confirmed','shipped','delivered','cancelled')),
  total_amount NUMERIC(12,2) NOT NULL,
  currency     CHAR(3) NOT NULL,
  metadata     JSONB NOT NULL DEFAULT '{}',
  version      INTEGER NOT NULL DEFAULT 0,          -- optimistic locking
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_orders_customer_status ON orders (customer_id, status);
CREATE INDEX ix_orders_created_at      ON orders (created_at);

CREATE TABLE order_items (
  id          UUID PRIMARY KEY,
  order_id    UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id  UUID NOT NULL,
  quantity    INTEGER NOT NULL CHECK (quantity > 0),
  unit_price  NUMERIC(12,2) NOT NULL CHECK (unit_price >= 0)
);
CREATE INDEX ix_order_items_order ON order_items (order_id);
```

Guidelines:
- **Let the database enforce what it can**: `NOT NULL`, `UNIQUE`, foreign keys, `CHECK`. Application validation is not a substitute — concurrent requests and other writers bypass it.
- **IDs**: UUIDs (prefer time-ordered UUIDv7 or ULID for index locality) let the domain create IDs before persisting; sequential integers are smaller and fine for internal tables. Don't expose guessable sequential IDs publicly where enumeration matters.
- **Money**: `NUMERIC`/`DECIMAL` or integer minor units, plus a currency column. Never `FLOAT`.
- **Time**: timezone-aware timestamps stored in UTC.
- **Enums**: `CHECK` constraints or lookup tables are easier to evolve than native enum types in some databases.
- **Index for your actual queries**: composite indexes in the order of equality filters then range/sort columns; index every foreign key you join or delete on. Too many indexes slow writes — measure.
- **Soft delete** (`deleted_at`) only when there's a real need (audit, undo); it complicates every query and unique constraint.
- **Separate schemas or databases per module/service** — a module's tables are private to it.

---

## Mapping Between Domain and Storage

Two legitimate approaches; pick per context:

| Approach | When | How |
|---|---|---|
| **Separate persistence models** (data mapper) | Rich domain, need persistence ignorance | Repository maps rows/ORM models ↔ domain entities (`toEntity`, `toRow`). Domain types have no ORM annotations. |
| **ORM entities are the domain model** (active record / annotated entities) | CRUD-heavy, simpler rules, framework-centric apps | Put behaviour on the ORM model; accept the coupling knowingly. |

```
// Data-mapper style repository using a query builder / raw SQL
class SqlOrderRepository(conn) implements OrderRepository {
  get(id) -> Order? {
    orderRow = conn.queryOne("SELECT * FROM orders WHERE id = $1", [id]) ?? return none
    itemRows = conn.query("SELECT * FROM order_items WHERE order_id = $1", [id])
    return OrderMapper.toEntity(orderRow, itemRows)
  }
}
```

---

## Connections and Transactions

- **Use a connection pool** sized to the workload: roughly `(DB cores × 2) + effective spindle count` across *all* app instances is a common starting point — many small pools from many instances can exhaust the database. Use a proxy pooler (PgBouncer, RDS Proxy) when instance counts are high or serverless.
- **Configure**: max size, acquire timeout, idle timeout, max lifetime (recycle), and a health/ping check.
- **Scope transactions to a unit of work** — short, and never spanning network calls to other services. See [architecture-patterns.md](architecture-patterns.md#unit-of-work-pattern).
- **Request-scoped session pattern**:

```
withTransaction(pool, work) {
  conn = pool.acquire(timeout = 5 seconds)
  try {
    conn.begin()
    result = work(conn)
    conn.commit()
    return result
  } catch error {
    conn.rollback()
    rethrow error
  } finally {
    pool.release(conn)
  }
}
```

- **Tests** use a separate database (or containerised instance) and isolate tests via per-test transactions rolled back at the end, or truncation between tests.

---

## Query Optimization

### N+1 queries

The most common ORM performance bug: loading a list, then lazily loading each child.

```
// ❌ N+1: 1 query for orders + N queries for items
orders = query("SELECT * FROM orders WHERE customer_id = $1", [customerId])
for o in orders: o.items = query("SELECT * FROM order_items WHERE order_id = $1", [o.id])

// ✅ Batched: 2 queries total
orders = query("SELECT * FROM orders WHERE customer_id = $1", [customerId])
items  = query("SELECT * FROM order_items WHERE order_id = ANY($1)", [ids(orders)])
attach(items, to = orders, by = order_id)

// ✅ Or a join when the child set is small and you need one round trip
query("SELECT o.*, i.* FROM orders o LEFT JOIN order_items i ON i.order_id = o.id WHERE o.id = $1", [id])
```

ORM equivalents: eager loading / `include` / `select_related`+`prefetch_related` / `selectinload` / `JOIN FETCH` / `Include()` / `preload` / `Repo.preload`. Turn on query logging in development and add tests that assert query counts for hot paths.

### Pagination

```sql
-- Keyset (cursor) pagination — uses the index, stable under inserts
SELECT * FROM orders
WHERE customer_id = $1 AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Offset pagination — simple, but cost grows with the offset
SELECT * FROM orders WHERE customer_id = $1 ORDER BY created_at DESC LIMIT 20 OFFSET 40;
```

Avoid `COUNT(*)` on every page request for large tables; return `hasMore` or cache approximate counts.

### Other habits

- **Select only needed columns** for list views; build read-optimised queries for the query side (see CQRS in [architecture-patterns.md](architecture-patterns.md#cqrs-light)).
- **Bulk operations** for batch writes (multi-row `INSERT`, `COPY`, batch APIs) instead of row-by-row loops.
- **Read `EXPLAIN (ANALYZE)`** for slow queries; fix with indexes or query shape before adding caches.
- **Always parameterise** — never concatenate user input into SQL. See [security.md](security.md#injection-prevention).
- **Read replicas** for heavy read traffic, remembering replica lag (read-your-own-writes may need the primary).

---

## Concurrency Control

```sql
-- Optimistic locking: update only if nobody else changed the row
UPDATE orders
SET status = 'confirmed', version = version + 1, updated_at = now()
WHERE id = $1 AND version = $2;
-- 0 rows affected  => concurrent modification -> fail Conflict, client retries/reloads
```

- **Optimistic** (version column) for typical user-facing edits with low contention.
- **Pessimistic** (`SELECT … FOR UPDATE`, ideally with `SKIP LOCKED` for job queues) for short critical sections with high contention.
- **Atomic statements** for counters and stock: `UPDATE inventory SET available = available - $2 WHERE product_id = $1 AND available >= $2` — check the affected row count instead of read-then-write.
- Know your isolation level (Read Committed is the common default); raise it only where anomalies matter.

---

## Migrations

Use a versioned migration tool (Flyway, Liquibase, EF Core, Prisma Migrate, Drizzle Kit, Alembic, Django, golang-migrate, goose, Atlas, sqlx, Diesel, Rails, Laravel, Doctrine, Ecto, …).

```sql
-- V2024_06_01_1200__add_orders_status_created_index.sql
-- up
CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_orders_status_created ON orders (status, created_at);
-- down
DROP INDEX CONCURRENTLY IF EXISTS ix_orders_status_created;
```

`CONCURRENTLY` can't run inside a transaction, and many migration tools wrap each migration in one — mark such migrations as non-transactional using the tool's option for that.

Rules for zero-downtime deploys (old and new code run side by side during a rollout):
1. **Expand → migrate → contract.** Add the new column/table (nullable or with a default) → deploy code that writes both and reads new → backfill in batches → deploy code that only uses new → drop the old column in a later release.
2. **Never rename or drop in the same release** as the code change that stops using it.
3. **Avoid long locks**: build indexes concurrently/online, add constraints as `NOT VALID` then validate, batch large backfills.
4. **Migrations are code**: reviewed, tested against a production-like copy, run automatically in CI/CD before (or as part of) deployment, and never edited after being applied.
5. **Seed data** for development lives in a separate script, not in migrations.
