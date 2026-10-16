# Event-Driven Architecture

Pseudocode conventions are in [SKILL.md](../SKILL.md#pseudocode-conventions). Broker examples (Kafka, RabbitMQ, cloud queues) are interchangeable roles; client libraries exist in every major language.

## Table of Contents

1. [Delivery Semantics](#delivery-semantics)
2. [Event Bus Port](#event-bus-port)
3. [Transactional Outbox](#transactional-outbox)
4. [Event Handlers and Idempotency](#event-handlers-and-idempotency)
5. [Retries and Dead-Letter Queues](#retries-and-dead-letter-queues)
6. [Saga Pattern](#saga-pattern)
7. [Event Schemas and Versioning](#event-schemas-and-versioning)
8. [Event Sourcing (Briefly)](#event-sourcing-briefly)

---

## Delivery Semantics

- **At-most-once**: may lose messages. Rarely acceptable for business events.
- **At-least-once**: may deliver duplicates. **The practical default** — pair it with idempotent consumers.
- **Exactly-once**: only within narrow boundaries (e.g. Kafka transactions inside Kafka). End-to-end, design for at-least-once + idempotency.

**Ordering** is usually guaranteed only per partition/queue/key. Partition by aggregate ID if events for one entity must be processed in order.

| Broker role | Examples | Strengths |
|---|---|---|
| Log / stream | Kafka, Redpanda, Kinesis, Pulsar | Replay, high throughput, ordered partitions, multiple consumer groups |
| Queue / router | RabbitMQ, SQS, Azure Service Bus, NATS JetStream | Work distribution, routing, per-message ack, delayed/retry queues |
| In-process | An in-memory dispatcher | Decoupling modules inside one deployable; no durability |

---

## Event Bus Port

```
interface EventPublisher {
  publish(event: IntegrationEvent) -> void
}

interface EventSubscriber {
  subscribe(eventType: string, handler: (Envelope) -> void, group: string) -> void
}

record Envelope {
  eventId:       UUID
  eventType:     string        // "orders.order_placed"
  schemaVersion: int
  occurredAt:    Instant
  correlationId: string        // ties together everything caused by one user request
  causationId:   string?       // the message that caused this one
  key:           string        // partition / ordering key, e.g. orderId
  payload:       bytes         // JSON / Avro / Protobuf
}
```

### In-process implementation (modules in a monolith, tests)

```
class InMemoryEventBus implements EventPublisher, EventSubscriber {
  private handlers: Map<string, List<Handler>>

  subscribe(type, handler, group) { handlers[type].append(handler) }

  publish(event) {
    for handler in handlers[event.eventType]:
      try { handler(envelope(event)) }
      catch error { log.error("handler failed", eventType = event.eventType, error); metrics.increment("event_handler_failures") }
      // Failures are recorded, never silently ignored. For durability, use the outbox even in-process.
  }
}
```

### Broker implementation (sketch)

```
class BrokerEventPublisher(producer: BrokerProducer, topicFor: (string) -> string) implements EventPublisher {
  publish(event) {
    producer.send(
      topic   = topicFor(event.eventType),
      key     = event.key,
      value   = serialize(event.payload),
      headers = { eventId, eventType, schemaVersion, occurredAt, correlationId, traceparent },
      acks    = ALL)                           // wait for durable write
  }
}

class BrokerEventSubscriber(consumer: BrokerConsumer) implements EventSubscriber {
  subscribe(type, handler, group) {
    consumer.join(group).onMessage(msg -> {
      try {
        handler(toEnvelope(msg))
        msg.ack()                              // ack only AFTER successful handling
      } catch error {
        msg.nack()                             // redeliver per retry policy; DLQ after max attempts (see below)
      }
    })
  }
}
```

---

## Transactional Outbox

The dual-write problem: `db.commit()` then `broker.publish()` — if the process crashes between them, the event is lost; if you publish first and the commit fails, you've announced something that didn't happen.

**Outbox**: write the event to an `outbox` table **in the same transaction** as the state change, then relay it.

```sql
CREATE TABLE outbox (
  id            UUID PRIMARY KEY,
  aggregate_id  TEXT NOT NULL,
  event_type    TEXT NOT NULL,
  payload       JSONB NOT NULL,
  headers       JSONB NOT NULL DEFAULT '{}',
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  published_at  TIMESTAMPTZ
);
CREATE INDEX ix_outbox_unpublished ON outbox (created_at) WHERE published_at IS NULL;
```

```
// Use case (inside the unit of work)
transaction uow {
  order.confirm(clock)
  uow.orders.update(order)
  uow.outbox.add(order.pullEvents())
  uow.commit()
}

// Relay (separate worker / background task; safe to run several instances)
loop every 500ms {
  transaction db {
    batch = db.query("SELECT * FROM outbox WHERE published_at IS NULL
                      ORDER BY created_at LIMIT 100 FOR UPDATE SKIP LOCKED")
    for row in batch:
      publisher.publish(row)                         // may publish a duplicate if we crash after this…
      db.execute("UPDATE outbox SET published_at = now() WHERE id = $1", [row.id])
  }
}
// …which is fine because consumers are idempotent. Purge old published rows periodically.
```

The relay is the one place where a network call inside a database transaction is acceptable: the row lock (`FOR UPDATE SKIP LOCKED`) is what lets several relay instances run safely. Keep batches small and broker timeouts short so locks are held briefly. Strict per-aggregate ordering additionally requires a single relay per partition key (or publishing in `created_at` order per aggregate).

Alternatives: change data capture (Debezium reading the database log), or a framework that does this for you (MassTransit, NServiceBus, Axon, Spring Modulith, Wolverine, Oban + Ecto.Multi).

---

## Event Handlers and Idempotency

Because delivery is at-least-once, **every handler must be safe to run twice**.

```
class ReserveInventoryOnOrderPlaced(uowFactory: UnitOfWorkFactory) {
  handle(env: Envelope) {
    transaction uowFactory.begin() as uow {
      if uow.processedMessages.exists(env.eventId, handler = "reserve-inventory"): return   // duplicate — skip
      event = deserialize<OrderPlaced>(env.payload)

      match uow.inventory.reserve(event.orderId, event.items) {
        ok                    -> log.info("inventory reserved", orderId = event.orderId)
        InsufficientInventory -> uow.outbox.add(InventoryReservationFailed(event.orderId, reason = "insufficient_inventory"))
      }

      uow.processedMessages.record(env.eventId, handler = "reserve-inventory")   // same transaction as the effect
      uow.commit()
    }
  }
}

class SendWelcomeEmailOnUserRegistered(email: EmailSender) {
  handle(env) {
    event = deserialize<UserRegistered>(env.payload)
    email.sendWelcome(to = event.email, idempotencyKey = env.eventId)   // provider-side dedupe
  }
}

// Registration (composition root)
subscriber.subscribe("users.user_registered", SendWelcomeEmailOnUserRegistered(emailSender).handle, group = "notifications")
subscriber.subscribe("orders.order_placed",   ReserveInventoryOnOrderPlaced(uowFactory).handle, group = "inventory")
```

Idempotency techniques: a processed-message table (same transaction as the effect), natural idempotency (`UPSERT`, "set status = X"), conditional updates with versions, idempotency keys passed to third parties.

Handlers should be **small and single-purpose**; one handler failing must not block unrelated handlers. Don't swallow errors — let them trigger a retry.

---

## Retries and Dead-Letter Queues

```
on handler failure:
  if error is transient (timeout, 503, lock conflict) and attempts < MAX_ATTEMPTS:
    redeliver after backoff (e.g. 1s, 5s, 30s, 2m, 10m — with jitter)
  else:
    move to dead-letter queue/topic with error details
    alert; provide tooling to inspect, fix, and replay
```

- **Poison messages** (always fail — bad schema, bug) must not block the partition/queue forever.
- Distinguish **retryable** from **non-retryable** errors in the handler's error types.
- Make DLQ replay a first-class operational tool, not an afterthought.

---

## Saga Pattern

A saga coordinates a business transaction across services using **local transactions plus compensating actions**, since there's no distributed ACID transaction.

| Style | How | Use when |
|---|---|---|
| **Choreography** | Each service reacts to events and emits new ones | Few steps, simple flow |
| **Orchestration** | A coordinator tells each participant what to do and tracks state | Many steps, branching, need visibility |

### Orchestrated saga (persisted state machine)

```
enum SagaState { RUNNING, COMPENSATING, COMPLETED, FAILED }   // per-step progress lives in saga.completedSteps

record SagaStep { name, action: () -> void, compensation: () -> void }

class PlaceOrderSaga(sagaStore, orders, inventory, payments, notifications) {
  steps(cmd, sagaId) = [
    SagaStep("create_order",      () -> orders.createPending(sagaId, cmd),          () -> orders.cancel(sagaId)),
    SagaStep("reserve_inventory", () -> inventory.reserve(sagaId, cmd.items),       () -> inventory.release(sagaId)),
    SagaStep("capture_payment",   () -> payments.capture(sagaId, cmd.paymentRef),   () -> payments.refund(sagaId)),
    SagaStep("confirm_order",     () -> orders.confirm(sagaId),                     () -> none),
  ]

  run(cmd) {
    saga = sagaStore.createOrLoad(cmd.idempotencyKey)       // resumable after a crash
    if saga.state == COMPLETED: return                      // duplicate request
    if saga.state in [COMPENSATING, FAILED]:                // crashed mid-compensation, or already failed
      compensate(saga); fail SagaFailed(saga.id)
    for step in steps(cmd, saga.id) where step.name not in saga.completedSteps:
      try {
        step.action()                                       // each action is idempotent (uses saga.id as key)
        saga.markCompleted(step.name); sagaStore.save(saga)
      } catch error {
        saga.state = COMPENSATING; sagaStore.save(saga)
        compensate(saga)
        fail SagaFailed(saga.id, step.name, error)
      }
    saga.state = COMPLETED; sagaStore.save(saga)
    notifications.sendOrderConfirmation(saga.id)            // after completion, not a compensable step
  }

  compensate(saga) {                                         // idempotent: skips already-compensated steps
    for name in reverse(saga.completedSteps) where name not in saga.compensatedSteps:
      retryWithBackoff { step(name).compensation() }        // compensations must eventually succeed
      saga.markCompensated(name); sagaStore.save(saga)
    saga.state = FAILED; sagaStore.save(saga)
  }
}
```

Key rules:
- **Persist saga state** after each step so it can resume after crashes; don't keep it only in memory.
- **Every action and compensation is idempotent** (keyed by saga ID).
- **Compensations are semantic**, not rollbacks: a refund, a cancellation email. Some actions (an email already sent) can't be undone — order steps so irreversible ones come last.
- Consider a workflow engine (Temporal, Camunda/Zeebe, AWS Step Functions, Azure Durable Functions, Restate) instead of hand-rolling persistence, timers, and retries.

---

## Event Schemas and Versioning

- Define integration events in a schema language (JSON Schema, Avro, Protobuf) stored in a shared `contracts/` repo or a schema registry; generate types per language.
- **Additive changes only** within a version: add optional fields; never remove/rename/retype.
- **Breaking change** → new event type or version (`order_placed.v2`), publish both during migration.
- Include `eventId`, `eventType`, `schemaVersion`, `occurredAt`, and correlation/trace IDs in every envelope.
- Publish **facts** with enough data for typical consumers ("event-carried state transfer") or just IDs ("notification" — consumers call back). Choose consciously: fat events reduce coupling at runtime, thin events reduce schema coupling.

---

## Event Sourcing (Briefly)

Instead of storing current state, store the **sequence of events** and derive state by replaying them. Gains: full audit trail, temporal queries, rebuilding read models. Costs: schema evolution of old events, snapshotting, eventual consistency of projections, a steep learning curve. Use it for domains where history *is* the requirement (ledgers, compliance-heavy workflows), not as a default. Tools: EventStoreDB/Kurrent, Marten (.NET), Axon (JVM), Commanded (Elixir), or a Postgres events table.
