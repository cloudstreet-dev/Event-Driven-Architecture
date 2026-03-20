# Fundamental Patterns

Concepts are lovely. Patterns are how you actually build things. This chapter covers the foundational patterns of event-driven architecture — the recurring solutions to recurring problems that show up in every non-trivial EDA implementation. Some of these patterns are simple enough to implement in an afternoon. Others are career-defining rabbit holes. We will cover both.

---

## Publish/Subscribe — The Gateway Pattern

Publish/subscribe (pub/sub) is the simplest and most widely used event-driven pattern. A producer publishes events to a topic (or channel, or subject — the terminology varies by broker). Consumers subscribe to topics and receive events as they are published. The producer does not know who the consumers are. The consumers do not coordinate with each other.

```
┌──────────┐         ┌─────────┐         ┌────────────┐
│ Producer  │──event──▶│  Topic  │──event──▶│ Consumer A │
│          │         │         │──event──▶│ Consumer B │
│          │         │         │──event──▶│ Consumer C │
└──────────┘         └─────────┘         └────────────┘
```

### Fan-Out

Every subscriber gets a copy of every event. This is the default pub/sub behaviour and is useful when multiple services need to react to the same event independently. `OrderPlaced` fans out to the billing service, the inventory service, the analytics pipeline, and the notification service — all receiving the same event, all acting on it differently.

### Consumer Groups

When you need *competing consumers* — multiple instances of the same service sharing the workload — you use consumer groups (Kafka terminology) or competing consumers (RabbitMQ terminology). Events on a topic are distributed among group members so that each event is processed by exactly one member of the group.

```
┌─────────┐         ┌────────────────────────────┐
│  Topic   │──event──▶│  Consumer Group "billing"  │
│         │         │  ┌────────┐  ┌────────┐    │
│         │         │  │ inst-1 │  │ inst-2 │    │
│         │         │  └────────┘  └────────┘    │
└─────────┘         └────────────────────────────┘
```

This gives you horizontal scalability for consumers. Need to process events faster? Add more instances to the group.

### Topic Design

Topic design is underappreciated. A few principles:

- **One event type per topic** is the simplest model and the one we recommend for most cases. `orders.placed`, `orders.shipped`, `payments.processed`. It makes subscription, filtering, and schema management straightforward.
- **One topic per entity** (all events for orders go to `orders`) is simpler operationally but requires consumers to filter by event type, which adds complexity to every consumer.
- **Avoid mega-topics.** A single topic called `events` that carries every event in your system is technically possible and practically a nightmare. Consumers cannot subscribe selectively, schema management is impossible, and consumer lag on one event type affects all others.

The right granularity depends on your broker, your volume, and your team's preferences. But err on the side of more topics rather than fewer. It is easier to merge topics than to split them.

### When Pub/Sub Is Enough

For many systems, basic pub/sub is all you need. Events are published, consumers react, and life is good. You do not need event sourcing. You do not need CQRS. You need a topic and some subscribers.

The urge to over-engineer is strong in the EDA community. Resist it. If pub/sub solves your problem, declare victory and move on.

---

## Event Streaming vs Message Queuing

These terms are often used interchangeably, which is unfortunate because they describe fundamentally different architectures. The distinction determines your system's capabilities around replay, retention, and consumer independence.

### Message Queuing (Destructive Consumption)

In a traditional message queue (RabbitMQ, ActiveMQ, SQS), a message is consumed and then removed from the queue. Once a consumer acknowledges a message, it is gone. Other consumers cannot read it. You cannot replay it.

```
Producer → [msg-1, msg-2, msg-3] → Consumer
                                    ↓
                              msg-1 consumed
                              (deleted from queue)
```

**Characteristics:**
- Messages are transient — consumed once and discarded.
- Delivery semantics are per-message: acknowledged, rejected, or dead-lettered.
- Consumers compete for messages (one message → one consumer).
- No replay capability (by default; some brokers offer limited retention).
- Excellent for task distribution and work queues.

### Event Streaming (Non-Destructive Consumption)

In an event streaming system (Kafka, Pulsar, Redpanda, Kinesis), events are appended to a log and retained for a configurable period (or indefinitely). Consumers maintain their own position (offset) in the log. Multiple consumers can read the same events independently, at different speeds, from different positions.

```
Log: [evt-1] [evt-2] [evt-3] [evt-4] [evt-5]
                ↑                        ↑
          Consumer A              Consumer B
          (offset 2)              (offset 5)
```

**Characteristics:**
- Events are retained — the log is the source of truth.
- Consumers are independent — each tracks its own offset.
- Replay is native — reset the offset, reprocess the history.
- Multiple consumer groups can read the same events at different rates.
- The log is both a communication channel and a storage system.

### When Each Matters

**Use message queuing when:**
- You have task-based workloads (send an email, process a payment, generate a report).
- You need per-message routing logic (route to different queues based on headers).
- Message order is less important than delivery guarantees.
- You do not need replay.

**Use event streaming when:**
- Events are facts you want to retain (business events, audit logs, state changes).
- Multiple consumers need to process the same events independently.
- You need replay capability (rebuilding state, recovering from bugs, backfilling new consumers).
- You are building event sourcing, CQRS, or stream processing pipelines.
- Ordering within a partition matters.

**The grey zone:** Many modern brokers blur the line. RabbitMQ has added stream support. Kafka can be used for task distribution. But the core architectures are different, and choosing the wrong one for your use case leads to either unnecessary complexity or missing capabilities.

A common mistake is using Kafka as a task queue. It works, but you are fighting the abstraction. Kafka's strengths — retention, replay, multi-consumer — are irrelevant for "process this job once and forget about it." Conversely, using RabbitMQ when you need event replay is possible (with Streams) but unnatural.

Pick the tool that matches your use case. This is easier advice to give than to follow, which is why Part 2 of this book exists.

---

## Event Sourcing

Event sourcing is the pattern that makes architects' eyes light up and operations teams' eyes narrow. The idea is simple: instead of storing the current state of an entity, you store the sequence of events that led to that state. The current state is derived by replaying the events.

### The Core Idea

Traditional approach (state-based):
```
┌─────────────────────────────┐
│ orders table                │
│ ─────────────────────────── │
│ id: ord-7829                │
│ status: shipped             │
│ total: 149.97               │
│ updated_at: 2025-11-14      │
└─────────────────────────────┘
```

Event-sourced approach:
```
┌─────────────────────────────────────────┐
│ events for ord-7829                     │
│ ─────────────────────────────────────── │
│ 1. OrderPlaced    { total: 149.97 }     │
│ 2. PaymentProcessed { paymentId: ... }  │
│ 3. OrderConfirmed { }                   │
│ 4. ShipmentCreated { trackingId: ... }  │
│ 5. OrderShipped   { carrier: "DHL" }    │
└─────────────────────────────────────────┘

Current state = replay(events 1..5) → { status: "shipped", total: 149.97, ... }
```

The event store is an append-only log. You never update or delete events. To get the current state, you replay all events for an entity through a state-building function (sometimes called a *fold*, *reducer*, or *aggregate hydration*).

```
function buildOrderState(events):
    state = { status: "new" }

    for event in events:
        switch event.type:
            case "OrderPlaced":
                state.orderId = event.data.orderId
                state.total = event.data.total
                state.status = "placed"

            case "PaymentProcessed":
                state.paymentId = event.data.paymentId
                state.status = "paid"

            case "OrderShipped":
                state.carrier = event.data.carrier
                state.status = "shipped"

            case "OrderCancelled":
                state.status = "cancelled"
                state.cancelReason = event.data.reason

    return state
```

### Benefits

**Complete audit trail.** You know exactly what happened and when. Not "the order is shipped" but "the order was placed at 10:15, paid at 10:16, confirmed at 10:17, and shipped at 14:32." Regulated industries (finance, healthcare) love this.

**Temporal queries.** "What was the state of this order at 10:30?" Replay events up to that timestamp and you have the answer. Try doing that with a mutable database row.

**Bug recovery.** If a bug corrupted state, you can fix the replay logic and rebuild correct state from the event history. You cannot do this if you have been overwriting state in a database.

**Event-driven by nature.** The events are already there. Publishing them to a broker for other services to consume is trivial.

**Debugging.** When something goes wrong, you have the complete history. No more "the order is in a weird state and we don't know how it got there."

### Costs

Event sourcing is not free, and the costs are often underestimated.

**Replay performance.** An entity with 10,000 events takes time to rebuild. Snapshots (periodic saves of the current state, so you only replay events since the last snapshot) mitigate this but add complexity.

```
function getOrderState(orderId):
    snapshot = snapshotStore.getLatest(orderId)  // e.g., state at event 9500
    events = eventStore.getEventsAfter(orderId, snapshot.version)  // events 9501-10000
    state = snapshot.state

    for event in events:
        state = applyEvent(state, event)

    return state
```

**Schema evolution.** Your events are immutable, but your understanding of the domain evolves. What happens when you need to change the structure of an event type that has millions of historical instances? You need upcasting — transforming old event formats to new ones during replay. This is manageable but requires discipline. Chapter 4 covers this in detail.

**Storage.** You are storing every event that ever happened. For high-volume systems, this adds up. Compaction strategies exist (keeping only the latest event per key) but they undermine the "complete history" benefit.

**Complexity.** Event sourcing is a significant mental model shift. Developers accustomed to CRUD need to learn to think in terms of events and projections. This is a training and hiring cost.

**Query complexity.** "Show me all orders over $100 that were placed this week" is a trivial SQL query against a state table. Against an event store, it requires either a projection (see CQRS below) or a full scan of events, neither of which is simple.

### When Event Sourcing Is Overkill

Event sourcing is overkill when:

- Your domain does not benefit from temporal queries or audit trails.
- Your entities have simple lifecycles (created, maybe updated, maybe deleted).
- Your team is not prepared for the complexity.
- Your read patterns are complex and varied (event sourcing alone makes querying painful).

A user profile that is updated occasionally and queried frequently does not benefit from event sourcing. A financial ledger that must maintain a complete, auditable history absolutely does.

The most common mistake is adopting event sourcing system-wide. Most systems have a few aggregates that benefit from it (the ones with complex state machines and audit requirements) and many that do not. Apply it selectively.

---

## CQRS — Command Query Responsibility Segregation

CQRS separates the write model (how you accept and validate changes) from the read model (how you query and display data). In an event-driven context, this typically means:

1. Commands are validated and processed by the write model, which emits events.
2. Events are consumed by one or more read model projectors, which build queryable views.
3. Queries are served from the read models.

```
┌──────────┐   command   ┌─────────────┐   events   ┌──────────────┐
│  Client   │───────────▶│ Write Model │───────────▶│  Event Store │
│          │             │  (Domain)   │            │  / Broker    │
└──────────┘             └─────────────┘            └──────┬───────┘
                                                          │
     ┌────────────────────────────────────────────────────┘
     │                    │                    │
     ▼                    ▼                    ▼
┌──────────┐       ┌──────────┐       ┌──────────┐
│ Read Model│      │ Read Model│      │ Read Model│
│ (List)   │       │ (Detail) │      │ (Search) │
└──────────┘       └──────────┘       └──────────┘
     │                    │                    │
     ▼                    ▼                    ▼
┌──────────┐       ┌──────────┐       ┌──────────┐
│ Postgres │       │  Redis   │       │ Elastic  │
│          │       │          │       │ search   │
└──────────┘       └──────────┘       └──────────┘
```

### The Read Model Projection Pattern

A projection (or projector) is a function that consumes events and builds a read model — a denormalized, query-optimized view of the data. Each read model is tailored to a specific query pattern.

```
// Projector for the "order summary" read model
function projectOrderSummary(event):
    switch event.type:
        case "OrderPlaced":
            db.upsert("order_summaries", {
                orderId: event.data.orderId,
                customerName: event.data.customerName,
                total: event.data.total,
                status: "placed",
                placedAt: event.time
            })

        case "OrderShipped":
            db.update("order_summaries",
                where: { orderId: event.data.orderId },
                set: { status: "shipped", shippedAt: event.time }
            )

        case "OrderCancelled":
            db.update("order_summaries",
                where: { orderId: event.data.orderId },
                set: { status: "cancelled" }
            )
```

The read model can use whatever storage is optimal for the query pattern:

- **PostgreSQL** for complex joins and ad-hoc queries.
- **Redis** for low-latency key-value lookups.
- **Elasticsearch** for full-text search.
- **A flat file** for exports (seriously — sometimes a CSV updated by a projector is the right answer).

### Benefits of CQRS

**Independent scaling.** Reads and writes can be scaled independently. Most systems are read-heavy; CQRS lets you optimise and scale the read path without touching the write path.

**Optimised read models.** Instead of one normalised schema that serves all queries poorly, you have multiple denormalised schemas, each optimised for its specific query pattern. The "order list" view has exactly the columns it needs. The "order detail" view has different columns. The "order search" view uses a search engine.

**Polyglot persistence.** Different read models can use different storage technologies. This sounds like overengineering until you realise that serving full-text search from a relational database and serving key-value lookups from Elasticsearch are both terrible ideas.

### Costs of CQRS

**Eventual consistency.** The read model lags behind the write model. A user who places an order and immediately views their order list may not see the new order. This gap is typically milliseconds to seconds, but it exists, and your UI needs to handle it.

Common mitigation: after a write, redirect the user to a confirmation page that reads from the write model (or uses the data from the write response), not from the read model. By the time the user navigates to the order list, the projection has caught up.

**Projection complexity.** Each read model is a consumer that must correctly process every relevant event type. Bugs in projectors lead to incorrect read models, and the fix is to replay events and rebuild the projection — which requires the events to be retained (hello, event streaming).

**Operational overhead.** You are now maintaining multiple databases. Each needs monitoring, backup, and capacity planning. This is a real cost.

### CQRS Without Event Sourcing

CQRS does not require event sourcing. You can have a traditional database as your write model and use database triggers, CDC (change data capture), or application-level events to update read models. This gives you the read-model benefits without the full complexity of event sourcing.

Conversely, event sourcing without CQRS is possible but painful — querying an event store directly is slow and limiting.

The two patterns are complementary but independent. Use the combination that matches your needs.

---

## Sagas: Choreography vs Orchestration

A saga is a pattern for managing long-running business transactions that span multiple services. In a monolith, you would wrap the whole thing in a database transaction. In a distributed system, you cannot (distributed transactions exist but are a special circle of performance hell). Instead, each step in the saga is a local transaction, and if a step fails, you execute *compensating transactions* to undo the previous steps.

### The Problem

Place an order. Reserve inventory. Process payment. Create shipment. If the payment fails, you need to release the reserved inventory. If shipment creation fails, you need to refund the payment and release the inventory. Each service owns its own data. There is no global transaction coordinator.

### Choreography

In a choreography-based saga, there is no central coordinator. Each service listens for events and acts autonomously.

```
Order Service          Inventory Service       Payment Service
     │                       │                       │
     │── OrderPlaced ───────▶│                       │
     │                       │── InventoryReserved ─▶│
     │                       │                       │── PaymentProcessed ──▶
     │◀── OrderConfirmed ────│◀──────────────────────│
```

And when things go wrong:

```
Order Service          Inventory Service       Payment Service
     │                       │                       │
     │── OrderPlaced ───────▶│                       │
     │                       │── InventoryReserved ─▶│
     │                       │                       │── PaymentFailed ──▶
     │                       │◀── CompensateInventory│
     │◀── InventoryReleased ─│                       │
     │── OrderFailed ───────▶│                       │
```

**Advantages:**
- No single point of failure. No central coordinator that must be highly available.
- Loose coupling. Services react to events, not instructions.
- Easy to add new steps — just subscribe to the relevant events.

**Disadvantages:**
- The business process is implicit — it exists in the aggregate behaviour of all services, not in any one place. Understanding the complete saga requires reading the code of every participating service.
- Debugging a failed saga requires correlating events across multiple services (correlation IDs are essential here).
- Cyclic dependencies can emerge, where Service A reacts to Service B which reacts to Service A.
- Adding compensating logic to every service increases complexity across the board.

### Orchestration

In an orchestration-based saga, a central orchestrator (sometimes called a saga coordinator or process manager) directs the flow. It sends commands to services and listens for their responses.

```
                    Saga Orchestrator
                         │
                         │── ReserveInventory ──────▶ Inventory Service
                         │◀── InventoryReserved ─────│
                         │
                         │── ProcessPayment ────────▶ Payment Service
                         │◀── PaymentProcessed ──────│
                         │
                         │── CreateShipment ─────────▶ Shipping Service
                         │◀── ShipmentCreated ────────│
                         │
                         │── ConfirmOrder ───────────▶ Order Service
```

The orchestrator knows the complete flow. It maintains the state of the saga and handles compensations.

```
function handleSaga(orderId):
    saga = { orderId, status: "started", steps: [] }

    // Step 1: Reserve inventory
    send(command: "ReserveInventory", data: { orderId })
    response = await("InventoryReserved" or "InventoryReservationFailed")

    if response.type == "InventoryReservationFailed":
        saga.status = "failed"
        send(command: "RejectOrder", data: { orderId, reason: "no inventory" })
        return

    saga.steps.push("inventory_reserved")

    // Step 2: Process payment
    send(command: "ProcessPayment", data: { orderId })
    response = await("PaymentProcessed" or "PaymentFailed")

    if response.type == "PaymentFailed":
        saga.status = "compensating"
        send(command: "ReleaseInventory", data: { orderId })  // compensate step 1
        send(command: "RejectOrder", data: { orderId, reason: "payment failed" })
        return

    saga.steps.push("payment_processed")

    // Step 3: Confirm order
    send(command: "ConfirmOrder", data: { orderId })
    saga.status = "completed"
```

**Advantages:**
- The business process is explicit and readable. You can look at the orchestrator and understand the entire saga.
- Compensating logic is centralised. Easier to test and reason about.
- No risk of cyclic dependencies.
- The orchestrator can implement complex logic (retries, timeouts, parallel steps) in one place.

**Disadvantages:**
- The orchestrator is a single point of failure. If it goes down, in-flight sagas stall.
- The orchestrator has knowledge of all participating services, which is a form of coupling.
- Risk of the orchestrator becoming a "god service" that contains too much business logic.

### Choosing Between Them

**Use choreography when:**
- The saga is simple (2-3 steps).
- The participants are independently developed and deployed.
- You value autonomy and decoupling over process visibility.

**Use orchestration when:**
- The saga is complex (4+ steps, branching logic, parallel steps).
- You need clear visibility into the saga's state.
- Compensating transactions are complex and benefit from centralisation.
- You are in a regulated industry that requires process auditability.

Many real-world systems use both. Simple flows are choreographed; complex flows are orchestrated. This is not inconsistency — it is pragmatism.

---

## Event-Driven State Machines

A state machine defines the valid states an entity can be in and the transitions between them. In an event-driven system, events trigger state transitions.

```
                    ┌────────────┐
         OrderPlaced│            │
    ┌───────────────▶   Placed   │
    │               │            │
    │               └─────┬──────┘
    │                     │ PaymentProcessed
    │               ┌─────▼──────┐
    │               │            │
    │               │    Paid    │
    │               │            │
    │               └─────┬──────┘
    │                     │ OrderShipped
    │               ┌─────▼──────┐
    │               │            │
    │               │  Shipped   │
    │               │            │
    │               └─────┬──────┘
    │                     │ OrderDelivered
    │               ┌─────▼──────┐
    │               │            │
    │               │ Delivered  │
    │               │            │
    │               └────────────┘
    │
    │ OrderCancelled (from Placed or Paid)
    │               ┌────────────┐
    └──────────────▶│ Cancelled  │
                    └────────────┘
```

The state machine enforces business rules:

```
function handleEvent(currentState, event):
    switch (currentState, event.type):
        case ("placed", "PaymentProcessed"):
            return "paid"

        case ("paid", "OrderShipped"):
            return "shipped"

        case ("shipped", "OrderDelivered"):
            return "delivered"

        case ("placed", "OrderCancelled"):
            return "cancelled"

        case ("paid", "OrderCancelled"):
            return "cancelled"  // with refund compensation

        case ("shipped", "OrderCancelled"):
            reject("Cannot cancel a shipped order")

        default:
            reject("Invalid transition: " + currentState + " → " + event.type)
```

### Why State Machines Matter in EDA

Without explicit state machines, you end up with implicit state logic scattered across event handlers. "Can we ship this order?" turns into checking four different fields instead of asking "is the order in the `paid` state?" State machines make invariants explicit and transitions auditable.

They also make invalid states unrepresentable (or at least rejectable). An order cannot transition from "delivered" to "placed." The state machine enforces this. Without it, you are relying on every developer who touches the code to remember the rules.

State machines pair naturally with event sourcing: the event history *is* the transition history, and the current state is derived by replaying transitions through the state machine.

---

## The Outbox Pattern

The outbox pattern solves one of the most insidious problems in event-driven architecture: the dual-write problem.

### The Problem

Your service needs to update its database *and* publish an event. These are two separate operations. If the database write succeeds but the event publish fails, your database has the new state but no one else knows about it. If the event is published but the database write fails, everyone else thinks something happened that did not actually persist.

```
// This is broken
function placeOrder(order):
    database.save(order)           // Step 1: succeeds
    eventBroker.publish(OrderPlaced)  // Step 2: fails (broker is down)
    // Database has the order, but no event was published.
    // The rest of the system doesn't know the order exists.
```

You cannot solve this with a try-catch that rolls back the database on publish failure, because the publish might have succeeded from the broker's perspective even if your client timed out waiting for the acknowledgement. Welcome to distributed systems.

### The Solution

Instead of publishing the event directly, write it to an *outbox table* in the same database, in the same transaction as the business data.

```
function placeOrder(order):
    transaction:
        database.save(order)
        database.insertIntoOutbox({
            id: newId(),
            type: "OrderPlaced",
            payload: { orderId: order.id, total: order.total },
            createdAt: now(),
            published: false
        })
```

A separate process (the *outbox publisher* or *relay*) polls the outbox table and publishes events to the broker:

```
// Outbox relay (runs continuously)
function publishOutboxEvents():
    while true:
        events = database.query(
            "SELECT * FROM outbox WHERE published = false ORDER BY createdAt LIMIT 100"
        )

        for event in events:
            eventBroker.publish(event)
            database.update("outbox", { id: event.id }, { published: true })

        sleep(100ms)
```

Because the business data and the outbox entry are written in the same transaction, they are guaranteed to be consistent. Either both exist or neither does. The relay process handles eventually publishing the event, with at-least-once semantics (if it crashes after publishing but before marking the event as published, it will re-publish on restart — which is why consumers must be idempotent).

### Outbox Cleanup

The outbox table grows. You need a cleanup process that removes (or archives) published events after a retention period. This is typically a scheduled job:

```sql
DELETE FROM outbox WHERE published = true AND createdAt < NOW() - INTERVAL '7 days'
```

### Polling vs Log-Tailing

The polling approach (query the outbox table periodically) is simple but introduces latency — events are not published until the next poll cycle. For lower latency, you can use database log-tailing (Change Data Capture) to detect new outbox entries and publish them immediately. We cover CDC next.

### The Transactional Outbox in Practice

The outbox pattern is the standard recommendation for reliable event publishing from transactional systems. It is used in production at scale by organisations that have discovered the hard way that "publish then save" and "save then publish" are both broken.

The main drawback is that it ties your event publishing to your database technology. If your service does not use a relational database, you need an alternative (e.g., event sourcing, where the event store *is* the source of truth and events are published from the store).

---

## Change Data Capture (CDC)

Change Data Capture is the pattern of capturing changes from a database's transaction log and publishing them as events. Instead of the application being responsible for publishing events, the database's own change log becomes the event source.

### How It Works

Every database maintains a transaction log (write-ahead log in PostgreSQL, binlog in MySQL, oplog in MongoDB). CDC tools read this log and convert changes into events.

```
┌────────────┐    writes    ┌──────────────┐
│ Application│─────────────▶│   Database   │
│            │              │              │
└────────────┘              └──────┬───────┘
                                   │ transaction log
                            ┌──────▼───────┐
                            │  CDC Tool    │
                            │  (Debezium)  │
                            └──────┬───────┘
                                   │ events
                            ┌──────▼───────┐
                            │ Event Broker │
                            │   (Kafka)    │
                            └──────────────┘
```

### Debezium

Debezium is the dominant open-source CDC platform. It supports PostgreSQL, MySQL, MongoDB, SQL Server, Oracle, and others. It runs as a Kafka Connect connector and produces change events to Kafka topics.

A Debezium change event for a PostgreSQL table looks something like:

```json
{
  "before": {
    "id": 7829,
    "status": "placed",
    "total": 149.97
  },
  "after": {
    "id": 7829,
    "status": "shipped",
    "total": 149.97
  },
  "source": {
    "connector": "postgresql",
    "db": "orders",
    "table": "orders",
    "txId": 559,
    "lsn": 33692736
  },
  "op": "u",
  "ts_ms": 1700000133447
}
```

This includes both the before and after state, the source table, the transaction ID, and the log sequence number. It is a faithful representation of what the database saw.

### CDC vs Application-Level Events

CDC and application-level events solve different problems and are not interchangeable.

| Aspect | CDC | Application Events |
|--------|-----|--------------------|
| **Source** | Database transaction log | Application code |
| **Granularity** | Row-level changes | Business-level events |
| **Semantics** | "Row X changed from A to B" | "OrderShipped" |
| **Domain language** | Database schema language | Business domain language |
| **Coupling** | Consumers coupled to DB schema | Consumers coupled to event schema |
| **Completeness** | Captures ALL changes, including those from direct DB modifications | Only captures changes the application knows about |

### When to Use CDC

**Outbox relay.** CDC is an excellent way to implement the outbox pattern without polling. Write events to the outbox table, and let CDC pick them up and publish them to Kafka. This gives you low-latency event publishing with transactional guarantees.

**Legacy system integration.** You have a legacy system that writes to a database but does not publish events. CDC lets you capture those changes without modifying the legacy code. This is the "strangle fig" approach to migration: wrap the old system in events and gradually replace it.

**Data pipeline ingestion.** Streaming database changes into a data warehouse or data lake. This is CDC's original use case and remains one of its strongest.

**Audit logging.** Every change to a table, captured automatically, without relying on the application to remember to log it.

### When Not to Use CDC

**When you need business-level events.** A CDC event says "the `status` column changed from `placed` to `shipped`." An application-level event says "OrderShipped" and includes the carrier, tracking number, and estimated delivery date. If your consumers need business semantics, CDC alone is insufficient — you will need a transformation layer.

**When schema coupling is unacceptable.** CDC consumers are coupled to your database schema. If you rename a column, every CDC consumer breaks. Application-level events provide an abstraction layer between your internal schema and your public event contract.

**When database compatibility is uncertain.** CDC depends on database-specific features (logical replication in PostgreSQL, binlog in MySQL). If you might change databases, the CDC pipeline needs to change too.

### CDC + Outbox: The Best of Both Worlds

The most robust pattern combines CDC with the outbox:

1. Application writes business-level events (with domain semantics, proper naming, and a well-designed schema) to an outbox table.
2. CDC captures new outbox entries from the transaction log.
3. CDC publishes them to the event broker.

This gives you:
- Transactional consistency (outbox is written in the same transaction as the business data).
- Low latency (CDC detects changes in near-real-time, no polling).
- Business-level event semantics (the outbox contains properly designed events, not raw table changes).
- No polling overhead.

Debezium's outbox event router is purpose-built for this pattern.

---

## Patterns in Combination

These patterns do not exist in isolation. In practice, they combine:

- **Event sourcing + CQRS:** The event store is the write model. Projectors build read models from the event stream. This is the most common combination and the one most people mean when they say "event sourcing."
- **Sagas + outbox:** Each saga step writes its command/event to an outbox table, ensuring reliable publishing.
- **CDC + event streaming:** Database changes are captured by CDC and published to Kafka, where stream processors transform and enrich them.
- **Pub/sub + state machines:** Events trigger state transitions in downstream services, with the state machine enforcing valid transitions.

The art is knowing which patterns to apply where. Not every service needs event sourcing. Not every interaction needs a saga. Not every database change needs CDC. The worst event-driven architectures are the ones that apply every pattern everywhere, creating a system of such overwhelming complexity that no one can reason about it.

Start with pub/sub. Add complexity only when a specific problem demands it. Document *why* each pattern was chosen. Your future self — and the poor soul who inherits your system — will be grateful.

---

## Chapter Summary

The patterns in this chapter form the toolkit of event-driven architecture:

- **Pub/sub** is the foundation — simple, effective, and sufficient for many use cases.
- **Event streaming** (log-based, with retention) is fundamentally different from **message queuing** (destructive consumption). Choose based on whether you need replay.
- **Event sourcing** stores events as the source of truth. Powerful for audit, temporal queries, and bug recovery; expensive in complexity and operational cost. Apply selectively.
- **CQRS** separates reads from writes, enabling optimised read models. It complements event sourcing but does not require it.
- **Sagas** manage distributed transactions. Use **choreography** for simple flows, **orchestration** for complex ones, or both.
- **State machines** make transitions explicit and invalid states unrepresentable.
- **The outbox pattern** solves the dual-write problem with transactional guarantees.
- **CDC** captures database changes as events, enabling legacy integration, data pipelines, and low-latency outbox publishing.

Each pattern has costs. Apply them deliberately. The next chapter covers how to evolve the schemas of the events these patterns produce — because the only thing harder than designing an event schema is changing one.
