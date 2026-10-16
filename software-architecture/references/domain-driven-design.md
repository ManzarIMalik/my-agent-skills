# Domain-Driven Design

Pseudocode conventions are defined in [SKILL.md](../SKILL.md#pseudocode-conventions). Translate with [language-adaptation.md](language-adaptation.md).

## Table of Contents

1. [Strategic Design](#strategic-design)
2. [Entities](#entities)
3. [Value Objects](#value-objects)
4. [Aggregates](#aggregates)
5. [Domain Events](#domain-events)
6. [Domain Services and Factories](#domain-services-and-factories)

---

## Strategic Design

Tactical patterns (entities, aggregates) only pay off once the boundaries are right.

- **Ubiquitous language**: one vocabulary shared by domain experts and code. Names in code, APIs, and tables match what the business says.
- **Bounded contexts**: each context has its own model and language. "Product" in *Catalog* (descriptions, images) is a different model from "Product" in *Inventory* (stock levels, locations). Don't force one shared model across contexts.
- **Context mapping**: decide how contexts integrate — published events, a shared API, an **anti-corruption layer** (a translator that keeps a legacy or external model from leaking into yours).
- **Core vs. supporting vs. generic subdomains**: invest DDD effort in the core domain (your competitive advantage). Buy or keep simple the generic ones (auth, billing, email).

---

## Entities

Entities have **identity** that persists through state changes. Two entities are equal if their IDs are equal, regardless of other fields.

```
abstract class Entity<Id> {
  id: Id
  equals(other) -> bool { return other is Entity and other.id == this.id }
  hash()        -> int  { return hash(id) }
}

enum OrderStatus { PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED }

class Order extends Entity<OrderId> {
  customerId: CustomerId
  items:      List<OrderItem>   (private, mutated only through methods)
  status:     OrderStatus
  discount:   Discount?
  updatedAt:  Instant

  // Creation goes through a factory that enforces invariants
  static create(customerId, items, discount?, clock: Clock) -> Order {
    if items is empty: fail RuleViolated("order_must_have_items", "Order must have at least one item")
    order = Order(id = OrderId.new(), customerId, items, status = PENDING, discount, updatedAt = clock.now())
    order.events.append(OrderPlaced(orderId = order.id, customerId, total = order.total(), itemCount = items.size))
    return order   // events are collected via pullEvents() — see Aggregates below
  }

  subtotal() -> Money { return sum(item.total() for item in items) }
  total()    -> Money { return discount ? subtotal().minus(discount.amount) : subtotal() }

  confirm(clock: Clock) {
    if status != PENDING: fail RuleViolated("confirm_pending_only", "Cannot confirm order in status " + status)
    status = CONFIRMED
    updatedAt = clock.now()
  }

  cancel(clock: Clock) {
    if status in [SHIPPED, DELIVERED]: fail RuleViolated("cancel_before_shipping", "Cannot cancel order in status " + status)
    status = CANCELLED
    updatedAt = clock.now()
  }
}
```

Guidelines:
- **Behaviour lives on the entity** (`order.confirm()`), not in a service that sets `order.status = CONFIRMED`. Anaemic models scatter invariants across the codebase.
- **Inject time** (`Clock`) and ID generation instead of calling the system clock/random directly — this keeps tests deterministic. Use UTC timezone-aware timestamps.
- **Model state as a closed set** (enum / sum type). In languages with rich types, consider a typestate or one type per state so invalid transitions don't compile.
- In functional languages, `confirm` is a pure function `confirm(order, now) -> Result<Order, Error>` returning a new value.

---

## Value Objects

Value objects are **immutable**, **compared by value**, and **validate themselves on construction** — so if you have one, it's valid.

```
record Email(value: string) {
  constructor validate {
    value = lowercase(trim(value))   // normalise first, once, here…
    if not matches(value, EMAIL_PATTERN): fail InvalidInput("Invalid email address: " + value)   // …then validate
  }
  toString() -> string { return value }
}

record Money(amount: Decimal, currency: CurrencyCode) {
  constructor validate {
    if currency.length != 3: fail InvalidInput("Invalid currency code: " + currency)
  }
  plus(other: Money) -> Money {
    if other.currency != currency: fail InvalidInput("Cannot add " + currency + " and " + other.currency)
    return Money(amount + other.amount, currency)
  }
  minus(other: Money) -> Money { ... same currency check ... }
  times(factor: Decimal) -> Money { return Money(round(amount × factor, currency.minorUnits), currency) }
  static zero(currency) -> Money { return Money(0, currency) }
}

record Address(street, city, region, postalCode, country) {
  constructor validate {
    if any field is blank: fail InvalidInput("All address fields are required")
  }
}

record Discount(amount: Money, reason: string) {
  constructor validate { if amount.amount < 0: fail InvalidInput("Discount cannot be negative") }
}
```

Guidelines:
- **Replace primitives** that carry rules: `Email`, `Money`, `Quantity`, `Percentage`, `DateRange`, typed IDs.
- **Money**: exact decimal or integer minor units; always carry currency. Decide whether negative amounts are allowed (a refund may be negative; a price usually isn't) — don't bake one rule into a type used for both.
- **Email validation**: a simple structural check plus a verification email beats an elaborate regex.
- To "change" a value object, create a new one.
- Language mapping: records/data classes, frozen dataclasses, readonly structs, Rust structs with private fields and a constructor returning `Result`, Go structs with unexported fields and a `NewX(...) (X, error)` constructor.

---

## Aggregates

An aggregate is a cluster of entities and value objects treated as **one consistency boundary**. External code holds references only to the **aggregate root** and changes the inside only through it.

```
class Order extends Entity<OrderId> {      // aggregate root
  private items:  List<OrderItem>
  private events: List<DomainEvent>
  status: OrderStatus

  items() -> ReadOnlyList<OrderItem> { return readOnlyView(items) }

  addItem(productId, quantity: Quantity, unitPrice: Money) {
    if status != PENDING: fail RuleViolated("modify_pending_only", "Can only add items to pending orders")
    if items.size >= MAX_ITEMS_PER_ORDER: fail RuleViolated("max_items", "Order cannot have more than " + MAX_ITEMS_PER_ORDER + " items")
    items.append(OrderItem.create(productId, quantity, unitPrice))
  }

  removeItem(itemId) {
    if status != PENDING: fail RuleViolated("modify_pending_only", "Can only remove items from pending orders")
    items.removeWhere(i -> i.id == itemId)
  }

  confirm(clock) {
    if status != PENDING: fail RuleViolated("confirm_pending_only", "Cannot confirm order in status " + status)
    if items is empty:    fail RuleViolated("confirm_non_empty", "Cannot confirm order without items")
    status = CONFIRMED
    events.append(OrderConfirmed(orderId = id, customerId = customerId, occurredAt = clock.now()))
  }

  pullEvents() -> List<DomainEvent> {   // repository/UoW collects these on save
    out = copy(events); events.clear(); return out
  }
}

class OrderItem extends Entity<OrderItemId> {   // only reachable through Order
  productId: ProductId
  quantity:  Quantity      // value object: positive integer
  unitPrice: Money
  total() -> Money { return unitPrice.times(quantity.value) }

  static create(productId, quantity, unitPrice) -> OrderItem {
    return OrderItem(id = OrderItemId.new(), productId, quantity, unitPrice)
  }
}
```

Aggregate design rules:
1. **Protect invariants inside the boundary** — anything that must be consistent *immediately* belongs in one aggregate.
2. **Keep aggregates small.** Big aggregates cause lock contention and slow loads.
3. **Reference other aggregates by ID**, not by object (`customerId`, not `customer`).
4. **One aggregate per transaction** is the ideal; changes to other aggregates happen via domain events (eventual consistency). Break this rule consciously, not by accident.
5. **Use optimistic concurrency** (a version number checked on update) so two concurrent writers can't silently overwrite each other.

---

## Domain Events

A domain event records **something that happened** that domain experts care about. Name in the past tense.

```
record DomainEvent {
  eventId:    UUID          // unique, for idempotent handling
  occurredAt: Instant       // UTC
  // eventType derived from the type name, or an explicit string constant
}

record OrderPlaced    extends DomainEvent { orderId, customerId, total: Money, itemCount: int }
record OrderConfirmed extends DomainEvent { orderId, customerId }
record OrderCancelled extends DomainEvent { orderId, customerId, reason: string }
record UserRegistered extends DomainEvent { userId, email }
record PaymentCaptured extends DomainEvent { paymentId, orderId, amount: Money }
```

Guidelines:
- **Raise events from aggregate methods**, collect them when saving, and publish via the outbox — see [event-driven.md](event-driven.md#transactional-outbox).
- **Events are immutable facts.** Never modify a published event; publish a new one or a new version.
- **Distinguish internal domain events from integration events.** Internal events can carry rich domain types; integration events crossing service boundaries are a public contract with a versioned schema and only the data consumers need.
- **Don't put personal data in events unnecessarily** — they're often retained for a long time.

---

## Domain Services and Factories

- **Domain service**: stateless business logic that involves several aggregates or doesn't naturally belong to one (`PricingService`, `TransferService`). Lives in the domain layer; takes and returns domain objects; does no I/O.
- **Factory**: encapsulates complex creation (`Order.create(...)`, `InvoiceFactory.fromOrder(order)`), ensuring every new object starts valid.
- **Specification** (optional): a reusable, named business predicate (`OverdueInvoiceSpec.isSatisfiedBy(invoice)`) that can also be translated into a query.
