# Architecture Patterns

Pseudocode conventions are defined in [SKILL.md](../SKILL.md#pseudocode-conventions). Translate with [language-adaptation.md](language-adaptation.md).

## Table of Contents

1. [Dependency Injection](#dependency-injection)
2. [Composition Root](#composition-root)
3. [Repository Pattern](#repository-pattern)
4. [Unit of Work Pattern](#unit-of-work-pattern)
5. [Service Layer Pattern](#service-layer-pattern)
6. [CQRS (Light)](#cqrs-light)
7. [When Not to Use These](#when-not-to-use-these)

---

## Dependency Injection

The core move: a component **receives** its collaborators through its constructor (or function parameters) instead of creating them. The component depends on an **interface (port)**; infrastructure supplies an **implementation (adapter)**.

### Define the Port (application layer)

```
interface UserRepository {
  getById(id: UserId) -> User?
  getByEmail(email: Email) -> User?
  save(user: User) -> void
  delete(id: UserId) -> void
}
```

Keep ports small and shaped by what the *consumer* needs, not by what the database can do. In Go, declare the interface in the consuming package; in languages with structural typing (TS, Go, Python `Protocol`) an implementation doesn't even need to name the interface.

### Implement the Adapter (infrastructure layer)

```
class SqlUserRepository(db: DatabaseConnection) implements UserRepository {
  getById(id) -> User? {
    row = db.queryOne("SELECT id, email, name, created_at FROM users WHERE id = $1", [id])
    return row ? toEntity(row) : none
  }

  save(user) {
    db.execute(
      "INSERT INTO users (id, email, name, created_at) VALUES ($1, $2, $3, $4)
       ON CONFLICT (id) DO UPDATE SET email = EXCLUDED.email, name = EXCLUDED.name",
      [user.id, user.email.value, user.name, user.createdAt])
  }

  // Mapping lives in the adapter so the domain never sees rows/ORM models.
  private toEntity(row) -> User {
    return User(id = row.id, email = Email(row.email), name = row.name, createdAt = row.created_at)
  }
}
```

With an ORM, the adapter maps between ORM models and domain entities. In frameworks where the ORM model *is* the domain model (Active Record style), that mapping disappears — acceptable for simpler domains.

### Use Case with Injected Dependencies

```
class GetUserUseCase(users: UserRepository) {
  execute(id: UserId) -> UserDTO {
    user = users.getById(id)
    if user is none: fail NotFound("User", id)
    return UserDTO.from(user)
  }
}
```

Note: cross-cutting concerns like caching are better added as a decorator around the repository than inside the use case — see [caching.md](caching.md#cached-repository-decorator).

---

## Composition Root

All concrete wiring happens in **one place** at application startup (`main`, `Program`, `bootstrap`, the DI container's module definitions). Everything else receives dependencies.

```
main() {
  config   = loadConfig()                         // from env/files, validated at startup
  dbPool   = Database.connect(config.databaseUrl, poolSize = config.dbPoolSize)
  cache    = RedisCache(config.redisUrl)

  users    = CachedUserRepository(SqlUserRepository(dbPool), cache, ttl = 5 minutes)
  getUser  = GetUserUseCase(users)

  server   = HttpServer(routes = [UserRoutes(getUser)])
  server.start(config.port)
  onShutdownSignal { server.drain(); dbPool.close(); cache.close() }
}
```

Using a DI container (Spring, ASP.NET Core, NestJS, Laravel, Dagger/Hilt, …) is the same idea with registration instead of manual construction. Pay attention to **lifetimes**: singletons for stateless services and pools, request/scoped for things like a unit of work or the current user.

---

## Repository Pattern

A repository presents a collection-like interface over persistence for **one aggregate type**.

### Aggregate-Specific Interface (preferred)

```
interface OrderRepository {
  get(id: OrderId) -> Order?
  add(order: Order) -> void
  update(order: Order) -> void            // or rely on the unit of work's change tracking
  findByCustomer(id: CustomerId, page: PageRequest) -> Page<Order>
  findPendingOlderThan(age: Duration) -> List<Order>
}
```

Name query methods in domain terms (`findPendingOlderThan`) rather than exposing a generic `find(criteria)` that leaks query semantics into the application layer.

### Generic Base (optional)

```
interface Repository<T, Id> {
  get(id: Id) -> T?
  add(entity: T) -> void
  remove(id: Id) -> void
  exists(id: Id) -> bool
}
interface OrderRepository extends Repository<Order, OrderId> { … }
```

Useful in languages with good generics and many similar aggregates (Java, C#, TS, Kotlin). Skip it in Go/Elixir/Ruby, and don't build one on top of an ORM that already gives you a generic repository (Spring Data, EF Core `DbSet`).

### Implementation Concerns

- **Load the whole aggregate** the domain needs in a bounded number of queries — avoid N+1 by eager-loading children (join or batched "IN" query). See [database-patterns.md](database-patterns.md#query-optimization).
- **Return domain objects, not rows or ORM entities.**
- **Don't put business rules in repositories** — they persist and retrieve; the domain decides.

---

## Unit of Work Pattern

A unit of work groups changes to several repositories into **one transaction**.

### Interface

```
interface UnitOfWork {
  users:    UserRepository
  orders:   OrderRepository
  payments: PaymentRepository
  outbox:   OutboxRepository
  commit() -> void
  rollback() -> void
}
interface UnitOfWorkFactory { begin() -> UnitOfWork }
```

### Implementation

```
class SqlUnitOfWork(conn: DatabaseConnection) implements UnitOfWork {
  init      { conn.beginTransaction() }
  users     = SqlUserRepository(conn)      // all repositories share the same connection/transaction
  orders    = SqlOrderRepository(conn)
  payments  = SqlPaymentRepository(conn)
  outbox    = SqlOutboxRepository(conn)
  commit()   { conn.commit() }
  rollback() { conn.rollback() }
  close()    { if not committed: rollback(); conn.release() }   // always runs (finally/defer/Drop/using)
}
```

### Usage

```
class PlaceOrderUseCase(uowFactory: UnitOfWorkFactory) {
  execute(cmd: PlaceOrderCommand) -> OrderDTO {
    transaction uowFactory.begin() as uow {
      customer = uow.users.getById(cmd.customerId)
      if customer is none: fail NotFound("User", cmd.customerId)

      order = Order.create(customer.id, cmd.items)
      uow.orders.add(order)
      uow.payments.add(Payment.pending(order.id, order.total))
      uow.outbox.add(order.pullEvents())   // events committed atomically with state

      uow.commit()
      return OrderDTO.from(order)
    }
  }
}
```

Do **not** call external services (payment gateways, emails, other microservices) inside the database transaction — the transaction can't roll them back, and holding locks while waiting on a network call hurts throughput. Commit first, then trigger side effects from the outbox. See [event-driven.md](event-driven.md#transactional-outbox).

Many frameworks provide this already (`@Transactional`, EF Core `DbContext.SaveChanges`, `Repo.transaction`/`Ecto.Multi`, Rails `transaction do`, Laravel `DB::transaction`). Use the built-in mechanism and just keep the *boundary* in the application layer.

---

## Service Layer Pattern

### Domain Service — pure business logic that doesn't belong to one entity

```
class PricingService {
  calculateDiscount(customer: Customer, items: List<OrderItem>, promo: PromoCode?) -> Discount {
    base       = sum(item.unitPrice × item.quantity for item in items)
    tierAmount = base × tierRate(customer.tier)
    promoAmount = promo ? promo.discountFor(base) : Money.zero(base.currency)

    return tierAmount >= promoAmount
      ? Discount(tierAmount, reason = "tier")
      : Discount(promoAmount, reason = "promo")
  }

  private tierRate(tier) -> Decimal {
    match tier { BRONZE -> 0.05, SILVER -> 0.10, GOLD -> 0.15, else -> 0 }
  }
}
```

No I/O, no framework — trivially unit-testable. Use exact decimal types for money (`BigDecimal`, `decimal`, `Decimal`, integer minor units), never binary floating point.

### Application Service — orchestration, no business rules

```
class OrderApplicationService(
  uowFactory: UnitOfWorkFactory,
  inventory:  InventoryPort,
  pricing:    PricingService
) {
  placeOrder(cmd: PlaceOrderCommand) -> OrderDTO {
    for item in cmd.items {
      available = inventory.availableQuantity(item.productId)
      if available < item.quantity:
        fail InsufficientInventory(item.productId, item.quantity, available)
    }

    transaction uowFactory.begin() as uow {
      customer = uow.users.getById(cmd.customerId) ?? fail NotFound("User", cmd.customerId)
      discount = pricing.calculateDiscount(customer, cmd.items, cmd.promoCode)
      order    = Order.create(customer.id, cmd.items, discount)

      uow.orders.add(order)
      uow.outbox.add(order.pullEvents())   // inventory reservation + confirmation email react to OrderPlaced
      uow.commit()
      return OrderDTO.from(order)
    }
  }
}
```

The availability check is an optimisation for fast feedback; the authoritative reservation must still be atomic (in the inventory store or via a saga), since stock can change between check and reserve.

---

## CQRS (Light)

Separate **commands** (change state, go through the domain model) from **queries** (read state, can bypass the domain and read optimised projections directly).

```
// Command side: full domain model, invariants enforced
PlaceOrderUseCase.execute(PlaceOrderCommand) -> OrderId

// Query side: thin, fast, shaped for the screen/API
interface OrderQueries {
  orderSummariesForCustomer(customerId, page) -> Page<OrderSummaryView>   // straight SQL/view/read model
}
```

Start with both sides on the same database. Move to separate read models (denormalised tables, search index) only when read and write needs genuinely diverge.

---

## When Not to Use These

- **Pure CRUD with no rules**: a handler calling a data-access layer (or the framework's ORM directly) is fine. Add layers when rules appear.
- **ORM already provides repository + unit of work** (EF Core, Hibernate, Ecto, Active Record): don't wrap it in an identical abstraction unless you need to keep the domain persistence-ignorant or swap stores.
- **Scripts, prototypes, one-off tools**: keep it simple; architecture is for code you'll maintain.
