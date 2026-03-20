# Core Concepts

Before you design your first event, before you choose a broker, before you argue with your team about whether Kafka is overkill — you need a shared vocabulary. This chapter establishes one. The distinctions here are not academic. Getting them wrong leads to architectures that look event-driven on the diagram but behave like a distributed monolith in production.

---

## Events vs Commands vs Queries

This is the foundational taxonomy. Every message flowing through your system is one of these three things, and conflating them is the single most common design mistake in event-driven systems.

### Events

An event is a notification that something happened. Past tense. Immutable. The producer is stating a fact about its own domain.

```
{
  "type": "InvoiceIssued",
  "source": "billing-service",
  "time": "2025-11-14T10:15:33Z",
  "data": {
    "invoiceId": "inv-9921",
    "customerId": "cust-441",
    "amount": 250.00,
    "currency": "EUR"
  }
}
```

Key properties:

- **Past tense.** `InvoiceIssued`, not `IssueInvoice`.
- **Owned by the producer.** The billing service decides what an `InvoiceIssued` event looks like. Consumers do not get a vote (though they get a voice via schema negotiation — see Chapter 4).
- **No expectation of a response.** The producer fires and forgets. It does not know or care who consumes the event.
- **Immutable.** Once published, an event is a historical fact. You do not update events; you publish new ones.

### Commands

A command is an instruction to do something. Imperative. Directed at a specific recipient. The sender *expects* something to happen as a result.

```
{
  "type": "SendWelcomeEmail",
  "target": "email-service",
  "data": {
    "recipientEmail": "alice@example.com",
    "templateId": "welcome-v2",
    "locale": "en-GB"
  }
}
```

Key properties:

- **Imperative.** `SendWelcomeEmail`, `ProcessRefund`, `ShipOrder`.
- **Directed.** There is one intended recipient. If you are broadcasting a command to "whoever wants to handle it," you have an event in disguise.
- **May fail.** The recipient may reject the command. The sender typically needs to know about this.
- **Has coupling.** The sender knows about the recipient and its capabilities. This is point-to-point messaging, not pub/sub.

### Queries

A query is a request for information. It does not change state. It is synchronous in nature, even when implemented over asynchronous transport.

```
{
  "type": "GetOrderStatus",
  "data": {
    "orderId": "ord-7829"
  }
}
```

Queries in an event-driven system are typically handled via request-response (HTTP, gRPC) rather than through the event broker. The async query pattern exists but adds complexity that is rarely justified. If you find yourself routing queries through Kafka, step back and ask what problem you are actually solving.

### Why the Distinction Matters

When you conflate events and commands, you end up with producers that *expect* consumers to act in specific ways. This recreates the coupling that event-driven architecture was supposed to eliminate. The producer starts to fail when the consumer does not behave as expected. You have reinvented RPC with extra steps and worse debugging.

A helpful litmus test: if the producer's correctness depends on what the consumer does with the message, you have a command, not an event. Design accordingly. Commands go to specific services over point-to-point channels. Events go to topics where anyone can subscribe.

---

## Anatomy of a Well-Designed Event

A surprising number of production incidents trace back to poorly designed events. Missing timestamps, absent correlation IDs, ambiguous types — these are not theoretical problems. They are the things that make your on-call engineers weep at 3 AM.

Here is what a well-designed event includes:

### Event ID

A globally unique identifier for this specific event instance. UUIDs (v4 or v7) are the standard choice. UUIDv7 is preferable where available because it is time-ordered, which makes log analysis and debugging easier.

```
"id": "01944b3c-8f3a-7d1e-a2b3-4c5d6e7f8901"
```

This ID serves multiple purposes:
- **Deduplication.** When a consumer receives the same event twice (and it will), the ID lets it recognise the duplicate.
- **Tracing.** You can follow a specific event through the system.
- **Idempotency keys.** Consumers can use the event ID to ensure they process each event exactly once.

Do not use auto-incrementing integers. They are not globally unique, they leak information about your event volume, and they create a coordination bottleneck.

### Timestamp

When the event occurred. Not when it was published, not when it was received — when the *thing that the event describes* happened. Use ISO 8601 format with timezone information. UTC is strongly preferred.

```
"time": "2025-11-14T10:15:33.447Z"
```

Include sub-second precision. You will need it for ordering, debugging, and performance analysis. Millisecond precision is the minimum; microsecond is better.

A word of caution: wall-clock timestamps are not reliable for ordering. Clocks drift between machines, NTP corrections can cause jumps, and two events that happened "at the same time" on different machines may have timestamps that suggest a different order. We discuss ordering properly later in this chapter. Use timestamps for human-readable debugging, not for determining causal order.

### Source

The identity of the system, service, or component that produced the event. This should be a stable identifier, not a hostname or IP address (which change with deployments).

```
"source": "billing-service"
```

or, following the CloudEvents URI convention:

```
"source": "/services/billing/eu-west-1"
```

### Event Type

A namespaced string that identifies what kind of event this is. Use a consistent naming convention across your organisation.

```
"type": "com.example.billing.InvoiceIssued"
```

Some conventions:
- Reverse domain notation: `com.example.billing.InvoiceIssued`
- Dot-separated hierarchy: `billing.invoice.issued`
- Simple PascalCase: `InvoiceIssued` (often sufficient for smaller systems)

Pick one. Enforce it. The naming convention matters less than consistency.

### Payload (Data)

The actual business data. What was ordered, how much was charged, which user signed up. This is the part that varies between event types.

```
"data": {
  "invoiceId": "inv-9921",
  "customerId": "cust-441",
  "lineItems": [
    { "sku": "WIDGET-42", "quantity": 3, "unitPrice": 49.99 },
    { "sku": "GADGET-7", "quantity": 1, "unitPrice": 100.03 }
  ],
  "totalAmount": 250.00,
  "currency": "EUR"
}
```

The payload design — what to include and what to omit — is one of the most consequential decisions in EDA. We will tackle this in the "Fat Events vs Thin Events" section below.

### Metadata

Non-business data about the event itself. This typically includes:

- **Schema version:** `"dataschema": "https://schemas.example.com/billing/invoice-issued/v2"`
- **Content type:** `"datacontenttype": "application/json"`
- **Correlation ID:** `"correlationid": "corr-abc-123"` (for tracing a business process across multiple events)
- **Causation ID:** `"causationid": "evt-xyz-789"` (the ID of the event that caused this one)

```
"metadata": {
  "schemaVersion": "2.1.0",
  "correlationId": "corr-abc-123",
  "causationId": "evt-xyz-789",
  "contentType": "application/json",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

### Correlation ID

This deserves special emphasis. A correlation ID is a unique identifier that follows a business process across multiple events and services. When a customer places an order, the initial `OrderPlaced` event gets a correlation ID. Every subsequent event in that order's lifecycle — `PaymentProcessed`, `InventoryReserved`, `ShipmentCreated` — carries the same correlation ID.

Without correlation IDs, debugging a multi-step business process in an event-driven system is an exercise in despair. You are searching through millions of events across dozens of services trying to reconstruct what happened to one order. With correlation IDs, you filter by a single value and get the complete story.

Make correlation IDs mandatory. Reject events that do not have them. You will thank yourself.

### Putting It All Together

A complete, well-designed event:

```json
{
  "specversion": "1.0",
  "id": "01944b3c-8f3a-7d1e-a2b3-4c5d6e7f8901",
  "type": "com.example.billing.InvoiceIssued",
  "source": "/services/billing/eu-west-1",
  "time": "2025-11-14T10:15:33.447Z",
  "datacontenttype": "application/json",
  "dataschema": "https://schemas.example.com/billing/invoice-issued/v2",
  "correlationid": "corr-abc-123",
  "causationid": "evt-xyz-789",
  "data": {
    "invoiceId": "inv-9921",
    "customerId": "cust-441",
    "totalAmount": 250.00,
    "currency": "EUR"
  }
}
```

You will notice this looks suspiciously like a CloudEvents envelope. That is not a coincidence. We will discuss CloudEvents formally later in this chapter.

---

## Event Taxonomy

Not all events are created equal, and not all events serve the same purpose. Understanding the different *kinds* of events will save you from shoehorning every use case into a single pattern.

### Domain Events

A domain event represents something meaningful that happened within a bounded context. It uses the language of the domain (the "ubiquitous language" if you are a Domain-Driven Design practitioner, which, in the context of EDA, you probably should be).

```
OrderPlaced
PaymentAuthorised
ShipmentDispatched
AccountSuspended
```

Domain events are the bread and butter of event-driven systems. They are raised by aggregates, published to event streams, and consumed by other bounded contexts. They describe *business facts*.

A well-designed domain event should be understandable by a domain expert, not just a developer. If your event is called `EntityStateTransition_V2`, you have abstracted away all the meaning.

### Integration Events

An integration event is a domain event that has been explicitly designed for consumption by other bounded contexts or external systems. The distinction matters because internal domain events may contain implementation details that should not leak across service boundaries.

For example, internally your billing service might raise a `InvoiceTaxCalculationCompleted` event with detailed tax breakdown data structures specific to your billing logic. The integration event published externally might be a simplified `InvoiceIssued` with just the total amount and tax summary.

Integration events form your public API. Treat them with the same care you would treat any public interface: version them, document them, and do not change them without warning.

### Notification Events

A notification event tells you that something happened but carries minimal data — just enough to identify what changed, not the details of the change.

```json
{
  "type": "OrderUpdated",
  "data": {
    "orderId": "ord-7829"
  }
}
```

The consumer, upon receiving this, must call back to the source service (via an API) to get the current state. This is the thinnest possible event, and it has a very specific use case: when the event data is large, changes frequently, and most consumers only care about the *latest* state.

The downside is obvious: you have reintroduced coupling. The consumer now depends on the producer's API being available. You have traded event payload size for runtime dependency. This is sometimes the right trade-off, but go in with your eyes open.

### Event-Carried State Transfer

The opposite of a notification event. An event-carried state transfer includes the complete current state of the entity in the event payload.

```json
{
  "type": "CustomerProfileUpdated",
  "data": {
    "customerId": "cust-441",
    "email": "alice@example.com",
    "name": "Alice Wonderland",
    "tier": "gold",
    "address": {
      "street": "42 Looking Glass Lane",
      "city": "Oxford",
      "postcode": "OX1 2JD",
      "country": "GB"
    },
    "preferences": {
      "newsletter": true,
      "smsNotifications": false
    }
  }
}
```

The consumer can build and maintain a local copy of the producer's data without ever calling the producer's API. This is full decoupling — no runtime dependency whatsoever. The consumer has everything it needs in the event.

The cost is larger events, more bandwidth, and the risk of stale data if events are delayed. It also means every consumer gets a full copy of data it might not need, which has privacy implications (does the shipping service really need the customer's newsletter preferences?).

---

## Fat Events vs Thin Events

This is one of the most debated design decisions in EDA, and the answer — annoyingly — is "it depends." But we can at least make the trade-offs explicit.

### Thin Events

A thin event contains the minimum information needed to identify what happened:

```json
{
  "type": "OrderPlaced",
  "data": {
    "orderId": "ord-7829"
  }
}
```

**Advantages:**
- Small payload, low bandwidth
- No risk of leaking data to consumers who should not have it
- Event schema rarely changes (there is not much to change)

**Disadvantages:**
- Consumers must call the producer's API to get details (coupling)
- The producer's API must handle the resulting query load
- If the producer is down, consumers are stuck
- You cannot replay thin events to rebuild state (the API state may have changed since the event was published)

### Fat Events

A fat event contains all the data a consumer could reasonably need:

```json
{
  "type": "OrderPlaced",
  "data": {
    "orderId": "ord-7829",
    "customerId": "cust-441",
    "customerEmail": "alice@example.com",
    "items": [
      { "sku": "WIDGET-42", "name": "Premium Widget", "quantity": 3, "unitPrice": 49.99 }
    ],
    "totalAmount": 149.97,
    "currency": "USD",
    "shippingAddress": { ... },
    "billingAddress": { ... },
    "placedAt": "2025-11-14T10:15:33Z"
  }
}
```

**Advantages:**
- True decoupling — consumers need nothing else
- Events are self-contained and replayable
- Consumers can build local read models without API calls
- Works even when the producer is offline

**Disadvantages:**
- Larger payloads, more bandwidth and storage
- Schema evolution is harder (more fields to manage)
- Risk of data leakage (every consumer gets every field)
- Event may include data the producer had to fetch from elsewhere, introducing latency at publish time

### The Pragmatic Middle Ground

In practice, most successful systems land somewhere in between: events include enough data for the *majority* of consumers to operate independently, while acknowledging that edge cases may require an API call. The common pattern is to include the key identifiers plus the data that changed:

```json
{
  "type": "OrderPlaced",
  "data": {
    "orderId": "ord-7829",
    "customerId": "cust-441",
    "items": [
      { "sku": "WIDGET-42", "quantity": 3, "unitPrice": 49.99 }
    ],
    "totalAmount": 149.97,
    "currency": "USD"
  }
}
```

The shipping address is not included because most consumers do not need it. The shipping service, which does need it, can call the order API. The analytics service, which just needs the amount and item count, has everything it needs in the event.

The guiding principle: **include data that most consumers need, exclude data that few consumers need, and always include enough to identify the entity and the change.**

---

## Event Envelopes and the CloudEvents Specification

An event envelope is the standard wrapper around your event data — the metadata fields that every event should carry, regardless of its business content. You can design your own, but there is a strong argument for adopting the CloudEvents specification.

### CloudEvents

CloudEvents is a CNCF (Cloud Native Computing Foundation) specification that defines a common structure for event metadata. Version 1.0 was released in 2019 and has since been adopted by most major cloud providers and many open-source projects.

The required attributes are:

| Attribute      | Type     | Description                                      |
|----------------|----------|--------------------------------------------------|
| `specversion`  | String   | CloudEvents spec version (currently `"1.0"`)     |
| `id`           | String   | Unique event identifier                          |
| `source`       | URI-ref  | Context in which the event happened              |
| `type`         | String   | Type of event                                    |

Optional but recommended attributes:

| Attribute           | Type      | Description                                 |
|---------------------|-----------|---------------------------------------------|
| `time`              | Timestamp | When the event occurred                     |
| `datacontenttype`   | String    | Content type of `data` (e.g., `application/json`) |
| `dataschema`        | URI       | Schema that `data` adheres to               |
| `subject`           | String    | Subject of the event in context of source   |

Extension attributes (you define these):

| Attribute        | Description                              |
|------------------|------------------------------------------|
| `correlationid`  | Business process correlation identifier  |
| `causationid`    | ID of the event that caused this one     |
| `partitionkey`   | Key for ordering/partitioning            |

### Why Adopt CloudEvents?

- **Interoperability.** If you ever need to integrate with cloud services, serverless functions, or third-party systems, CloudEvents is the lingua franca.
- **Tooling.** SDKs exist for every major language. Parsers, validators, and protocol bindings are available off the shelf.
- **Convention over invention.** Designing your own envelope means designing your own conventions, documenting them, building tooling for them, and training every new hire. CloudEvents has done this work for you.
- **Protocol bindings.** CloudEvents defines standard ways to map events onto HTTP, Kafka, AMQP, MQTT, and other transports. This removes an entire category of "how do we encode the metadata" debates.

### When Not to Adopt CloudEvents

If you are in a high-performance, low-latency context (financial trading, gaming, real-time telemetry), the overhead of JSON-encoded CloudEvents metadata may be unacceptable. In these environments, custom binary envelopes with Protocol Buffers, FlatBuffers, or SBE (Simple Binary Encoding) are common. The concepts are the same; the serialisation differs.

Also, if your system is entirely internal and you have strong existing conventions, migrating to CloudEvents may not be worth the disruption. But if you are starting from scratch, adopt CloudEvents. Seriously. You will not regret having a standard.

---

## Idempotency: Your Best Friend

In distributed systems, messages can be delivered more than once. Your broker might retry on timeout. Your consumer might crash after processing an event but before acknowledging it. A network partition might cause a producer to retry a publish. The result is the same: duplicate events.

Idempotency means that processing the same event multiple times produces the same result as processing it once. This is not optional in event-driven systems. It is a requirement.

### Why Duplicates Happen

Consider this sequence:

1. Consumer receives event `OrderPlaced` with ID `evt-123`.
2. Consumer processes the event (creates an invoice).
3. Consumer crashes before sending an acknowledgement to the broker.
4. Broker, having received no acknowledgement, redelivers `evt-123`.
5. Consumer receives the event again.

If the consumer naively processes the event again, the customer gets two invoices. This is not a theoretical concern — it is a Tuesday.

### Strategies for Idempotency

**1. Idempotency Key Table**

Maintain a table (or set) of processed event IDs. Before processing, check if the event ID has been seen:

```
function handleEvent(event):
    if eventStore.hasBeenProcessed(event.id):
        log("Duplicate event, skipping: " + event.id)
        return

    // Process the event
    processBusinessLogic(event)

    // Record that we've processed it
    eventStore.markAsProcessed(event.id)
```

The critical subtlety: the business logic and the `markAsProcessed` call must be in the same transaction. If they are not, you have a gap where a crash between the two leads to either lost events or duplicates.

```
function handleEvent(event):
    transaction:
        if eventStore.hasBeenProcessed(event.id):
            return

        processBusinessLogic(event)
        eventStore.markAsProcessed(event.id)
```

**2. Natural Idempotency**

Some operations are naturally idempotent. Setting a value is idempotent; incrementing a value is not.

```
// Idempotent: processing this twice has the same effect as once
UPDATE users SET email = 'alice@example.com' WHERE id = 'cust-441'

// NOT idempotent: processing this twice doubles the effect
UPDATE accounts SET balance = balance + 100 WHERE id = 'acct-992'
```

Where possible, design your event handlers to use naturally idempotent operations. Instead of "add $100 to the balance," use "set the balance to $1,350 as of event evt-123." This requires the event to carry enough state, which circles back to the fat-vs-thin debate.

**3. Conditional Writes**

Use optimistic concurrency control: only apply the change if the current state matches what you expect.

```
UPDATE orders
SET status = 'shipped', version = 4
WHERE id = 'ord-7829' AND version = 3
```

If the event is processed twice, the second attempt finds `version = 4` instead of `3`, the update affects zero rows, and the duplicate is harmlessly absorbed.

**4. Deduplication at the Broker Level**

Some brokers support producer-side deduplication (Kafka's idempotent producer, for example). This prevents duplicate *publishing* but does not protect against duplicate *consumption*. You still need consumer-side idempotency.

### The Idempotency Window

You cannot store every event ID forever. At some point, you need to prune the idempotency table. The question is: how long should you keep IDs?

This depends on your redelivery window. If your broker retries for up to 7 days, your idempotency table needs to retain IDs for at least 7 days (plus a safety margin). In practice, 14 to 30 days is common. After that, if a duplicate somehow arrives, you accept the vanishingly small risk.

For event-sourced systems, the idempotency window is effectively infinite — you have the full event history, and deduplication is inherent.

---

## Causality and Ordering

Ordering is the problem that makes distributed systems researchers write papers with titles like "Time, Clocks, and the Ordering of Events in a Distributed System" (Lamport, 1978). It is also the problem that makes practitioners swear at their screens when events arrive in the wrong order.

### The Fundamental Problem

In a distributed system, there is no single global clock. Two events that happen on different machines may have timestamps that suggest order A→B, when in fact the causal order was B→A (because one machine's clock was ahead). Wall-clock time is unreliable for ordering.

Even within a single machine, if events are published to different partitions of a topic, they may be consumed in a different order than they were produced. Ordering guarantees in most brokers are *per-partition*, not global.

### Wall-Clock Time

Despite its unreliability, wall-clock time (timestamps) is what most systems use for ordering. This works well enough when:

- Events are produced by the same service (clocks are likely synchronised within a cluster).
- Sub-second ordering precision is not required.
- NTP is configured and functioning on all machines.

It breaks down when:
- Events come from different services on different machines.
- Precise ordering matters (financial transactions, inventory counts).
- Clock skew exceeds your tolerance.

For most business applications, wall-clock timestamps with NTP synchronisation are "good enough." But you should know when they are not.

### Sequence Numbers

Within a single partition or stream, most brokers assign a monotonically increasing sequence number (Kafka calls it an *offset*, Pulsar calls it a *message ID*). This gives you a total order within a partition.

The trick is ensuring that causally related events end up in the same partition. The standard approach is to partition by an entity ID (e.g., order ID), so all events for a given order are in the same partition and thus totally ordered.

```
// Publishing with a partition key ensures ordering per-entity
producer.publish(
    topic: "orders",
    partitionKey: event.data.orderId,
    value: event
)
```

This works well for entity-level ordering. It does not help with ordering *across* entities ("did the payment happen before or after the inventory check?").

### Logical Clocks

A Lamport clock is a counter that each process maintains. The rules are simple:

1. Before sending a message, increment the counter and include it in the message.
2. Upon receiving a message, set your counter to `max(local, received) + 1`.

This gives you a partial order: if event A's Lamport timestamp is less than event B's, *and* there is a causal chain from A to B, then A happened before B. But if two events have no causal relationship, their Lamport timestamps tell you nothing about which happened first.

### Vector Clocks

Vector clocks extend Lamport clocks to capture the full causal history. Each process maintains a vector of counters, one per process. This allows you to determine whether two events are causally related or concurrent.

```
// Process A's vector clock after sending: [A:3, B:1, C:2]
// Process B's vector clock after receiving: [A:3, B:4, C:2]

// Comparing two vector clocks:
// [A:3, B:1, C:2] < [A:3, B:4, C:2]  → first causally precedes second
// [A:3, B:1, C:2] || [A:2, B:4, C:2]  → concurrent (neither precedes)
```

Vector clocks are elegant but have practical challenges:

- The vector grows with the number of processes. In a microservices system with hundreds of services, the overhead is significant.
- Garbage collection of vector clock entries is non-trivial.
- Most developers find them confusing (this is a statement about adoption feasibility, not developer intelligence).

In practice, vector clocks are used in databases (Dynamo, Riak) more than in application-level event systems. For most EDA use cases, partition-level ordering combined with entity-based partitioning is sufficient.

### Handling Out-of-Order Events

Regardless of your ordering strategy, consumers should be prepared for out-of-order delivery. The strategies are:

**1. Buffer and Reorder**

Collect events in a buffer, sort by sequence number or timestamp, and process in order. This adds latency and complexity but guarantees order.

```
function onEventReceived(event):
    buffer.add(event)

    while buffer.hasNextInSequence(lastProcessedSequence + 1):
        nextEvent = buffer.removeNextInSequence(lastProcessedSequence + 1)
        process(nextEvent)
        lastProcessedSequence = nextEvent.sequenceNumber
```

**2. Last-Write-Wins**

If you only care about the latest state, ignore events with a timestamp older than the last processed event for a given entity.

```
function onEventReceived(event):
    currentTimestamp = stateStore.getLastUpdated(event.entityId)
    if event.timestamp <= currentTimestamp:
        log("Stale event, skipping")
        return
    process(event)
    stateStore.setLastUpdated(event.entityId, event.timestamp)
```

This is simple but lossy — intermediate states are silently dropped.

**3. Version Checking**

Include a version number in your events. Only process an event if its version is exactly `currentVersion + 1`. If it is higher, buffer it. If it is lower, discard it.

**4. Accept the Chaos**

For some use cases — analytics, logging, non-critical notifications — ordering does not matter. An analytics dashboard that counts orders does not care whether `OrderPlaced` for order 100 arrives before or after `OrderPlaced` for order 101. If ordering does not affect correctness, do not pay the cost of enforcing it.

### The Pragmatic Summary

For most systems:

1. Partition by entity ID to get per-entity ordering.
2. Use correlation IDs and causation IDs to reconstruct causal chains during debugging.
3. Make consumers tolerant of out-of-order delivery where possible.
4. Reserve strict global ordering for the rare cases where it is genuinely required (and accept the throughput cost).

Trying to achieve strict global ordering across a distributed system is technically possible but operationally expensive. Usually, you do not need it. When you think you do, check twice.

---

## Chapter Summary

The concepts in this chapter are the vocabulary of event-driven architecture:

- **Events**, **commands**, and **queries** are fundamentally different things. Do not conflate them.
- A well-designed event has an **ID**, **timestamp**, **source**, **type**, **payload**, and **metadata** (including correlation and causation IDs).
- Events come in different flavours: **domain events**, **integration events**, **notification events**, and **event-carried state transfer**. Each has different trade-offs.
- The **fat vs thin** event debate is a trade-off between decoupling and payload size. Lean toward fat events unless you have a good reason not to.
- **CloudEvents** provides a standard envelope format. Adopt it unless you have specific reasons not to.
- **Idempotency** is not optional. Design every consumer to handle duplicate events correctly.
- **Ordering** is harder than it looks. Use partition-level ordering, entity-based partitioning, and accept that global ordering is usually not worth the cost.

With this vocabulary established, we can move on to the patterns that put these concepts to work.
