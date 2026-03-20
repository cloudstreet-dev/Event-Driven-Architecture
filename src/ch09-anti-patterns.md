# Anti-Patterns and Pitfalls

Every architectural style has its failure modes — the predictable ways that well-intentioned teams turn a good idea into a bad system. Event-driven architecture is no exception. In fact, because EDA is genuinely powerful and genuinely different from what most teams are used to, its failure modes are often more spectacular. A poorly designed monolith is slow. A poorly designed event-driven system is slow, inconsistent, impossible to debug, and occasionally loses data in ways that take weeks to discover.

This chapter is a catalog of the ways things go wrong. Some of these are mistakes you make during initial design. Others are diseases that develop gradually, like architectural arthritis, until one day the system can barely move. All of them are easier to prevent than to cure.

---

## Event Soup — When Everything Is an Event and Nothing Makes Sense

**The pattern:** The team discovers events and goes all-in. Every action, every state change, every internal implementation detail becomes an event. The system produces hundreds of event types, many of which are consumed by nobody, and the event stream becomes an incomprehensible torrent of noise.

**What it looks like:**

```
Topic: user-events
Events:
  UserLoggedIn
  UserLoggedOut
  UserClickedButton
  UserScrolledPage
  UserMovedMouse          <- really?
  UserHoveredOverLink
  UserResizedBrowser
  UserSessionHeartbeat
  UserPreferencesLoaded
  UserCacheWarmed
  UserDatabaseQueryExecuted  <- this is not a domain event
  UserThreadPoolAdjusted     <- this is an internal implementation detail
```

**Why it happens:** Teams conflate domain events with technical events. A domain event represents something meaningful that happened in the business domain — "a customer placed an order." A technical event represents something that happened inside a system — "the connection pool was resized." These are fundamentally different things and should not live in the same event stream or, in most cases, in an event stream at all.

**The damage:** Consumer teams can't find the events they care about amid the noise. Topic size grows rapidly, increasing storage costs and replay times. New developers stare at the event catalog and give up trying to understand it. Monitoring alerts fire on meaningless events.

**How to fix it:**

1. **Apply the "would a domain expert care?" test.** If you told a product manager about this event, would they care? "Customer placed an order" — yes. "Thread pool resized to 50" — no.
2. **Separate domain events from operational telemetry.** Operational data belongs in metrics, logs, and traces — not in domain event streams.
3. **Require at least one consumer before publishing an event.** If nobody consumes it, don't produce it. Events without consumers are just disk usage.
4. **Maintain an event catalog** with ownership, purpose, and known consumers for each event type.

---

## God Events — The 47-Field Event That Knows Too Much

**The pattern:** A single event type carries an enormous payload containing everything any consumer might ever need. Instead of publishing focused events that represent specific things that happened, the producer dumps its entire internal state into every event.

**What it looks like:**

```json
{
  "eventType": "OrderUpdated",
  "orderId": "ord-123",
  "customerId": "cust-456",
  "customerName": "Alice Smith",
  "customerEmail": "alice@example.com",
  "customerPhone": "+1-555-0123",
  "customerLoyaltyTier": "GOLD",
  "customerLifetimeValue": 12450.00,
  "customerSignupDate": "2019-03-15",
  "shippingStreet": "123 Main St",
  "shippingCity": "Springfield",
  "shippingState": "IL",
  "shippingZip": "62701",
  "shippingCountry": "US",
  "billingStreet": "456 Oak Ave",
  "billingCity": "Springfield",
  "billingState": "IL",
  "billingZip": "62702",
  "billingCountry": "US",
  "items": [...],
  "subtotal": 59.98,
  "taxRate": 0.0825,
  "taxAmount": 4.95,
  "shippingCost": 5.99,
  "totalAmount": 70.92,
  "currency": "USD",
  "paymentMethod": "CREDIT_CARD",
  "paymentLast4": "4242",
  "paymentBrand": "VISA",
  "warehouseId": "wh-east-1",
  "fulfillmentPriority": "STANDARD",
  "estimatedDelivery": "2025-11-20",
  "internalNotes": "Customer called about delivery window",
  "createdAt": "2025-11-15T10:30:00Z",
  "updatedAt": "2025-11-15T14:22:00Z",
  "updatedBy": "system",
  "previousStatus": "PENDING",
  "currentStatus": "CONFIRMED",
  "statusReason": "Payment verified",
  "version": 17
}
```

**Why it happens:** The producer team doesn't know what consumers need, so they include everything. Or a single generic event type like `OrderUpdated` replaces what should be multiple specific events (`OrderConfirmed`, `OrderShipped`, `OrderCancelled`).

**The damage:**
- **Tight coupling.** Every consumer is coupled to the producer's internal data model. When the producer renames `customerLoyaltyTier` to `loyaltyLevel`, every consumer breaks.
- **PII sprawl.** The event contains customer PII (name, email, phone, addresses) even when the consumer only needed the order status. Now every consumer of this topic has access to PII, regardless of whether they need it.
- **Schema evolution hell.** Evolving a 47-field schema is exponentially harder than evolving a 7-field schema. Every field is a potential breaking change.
- **Bandwidth waste.** The shipping service needs `orderId`, `shippingAddress`, and `items`. It receives 2KB of data it ignores.

**How to fix it:**

1. **Use specific event types** instead of generic ones. `OrderConfirmed` is better than `OrderUpdated` with `currentStatus: "CONFIRMED"`. Each event type carries only the fields relevant to what happened.
2. **Apply interface segregation.** An event should contain only the data needed to understand what happened. Consumers that need more context can look it up by ID.
3. **Separate the event from the entity.** An event is not a database row notification. It's a record of something that happened, with enough context to be meaningful but not so much that it's a data dump.

```json
// BETTER: Focused event
{
  "eventType": "OrderConfirmed",
  "orderId": "ord-123",
  "customerId": "cust-456",
  "confirmedAt": "2025-11-15T14:22:00Z",
  "totalAmount": "70.92",
  "currency": "USD",
  "estimatedDelivery": "2025-11-20"
}
```

---

## The Distributed Monolith — Temporal Coupling Through Events

**The pattern:** Services are technically separate, deployed independently, and communicate through events. But they're coupled so tightly through event dependencies that you can't change, deploy, or operate any of them independently. You've achieved the worst of both worlds: the complexity of distribution with none of the benefits.

**What it looks like:**

```
OrderService                                    (1) OrderPlaced →
  PaymentService                                (2) PaymentProcessed →
    InventoryService                            (3) InventoryReserved →
      ShippingService                           (4) ShipmentCreated →
        NotificationService                     (5) CustomerNotified →
          AnalyticsService                      (6) OrderAnalyticsUpdated →
            LoyaltyService                      (7) LoyaltyPointsAwarded
```

Every service waits for the previous service to complete before it can act. Changing the order of operations requires changing multiple services. A failure in step 3 cascades to steps 4-7. You have a sequential pipeline disguised as an event-driven architecture.

**The test:** Can you deploy `InventoryService` without coordinating with `PaymentService` and `ShippingService`? If not, you have a distributed monolith.

**Why it happens:** Teams take an existing sequential workflow and replace synchronous calls with events without rethinking the workflow. The arrows change from HTTP calls to events, but the dependencies don't.

**The damage:**
- **Deploy coupling.** Services must be deployed in a specific order. A schema change in `OrderPlaced` triggers a cascade of changes across every downstream service.
- **Fragile chains.** The reliability of the chain is the product of the reliability of each link. If each service is 99.9% reliable, a 7-service chain is 99.3% reliable — and that's before accounting for the broker.
- **Debugging nightmares.** An end-to-end operation touches 7 services. Finding where something went wrong requires correlating events across all of them.

**How to fix it:**

1. **Identify truly independent reactions.** In the example above, `ShippingService` genuinely needs to know the payment succeeded. But does `AnalyticsService` need to wait for `ShipmentCreated`? Can it react directly to `OrderPlaced`?
2. **Fan out, don't chain.** Multiple services reacting to the same event is good (fan-out). Services forming a daisy chain where each depends on the previous one's output is often bad (pipeline).
3. **Choreography doesn't mean sequential.** The beauty of event-driven choreography is that independent actions happen in parallel. If everything is serial, you might want an orchestrator (a saga) that coordinates explicitly.

```
// BETTER: Fan-out from the originating event
OrderService: OrderPlaced →
  ├── PaymentService (reacts to OrderPlaced)
  ├── InventoryService (reacts to OrderPlaced)
  ├── AnalyticsService (reacts to OrderPlaced)
  └── NotificationService (reacts to OrderPlaced)

PaymentService: PaymentProcessed →
  ├── ShippingService (reacts to PaymentProcessed + InventoryReserved)
  └── LoyaltyService (reacts to PaymentProcessed)
```

---

## Chatty Services — Death by a Thousand Events

**The pattern:** Services communicate every minor internal state change as an event. A single user action generates dozens of events, each triggering further processing, which generates more events, which triggers more processing. The system is drowning in its own verbosity.

**What it looks like:**

User clicks "Place Order" -> the system produces:

```
OrderInitiated
OrderValidationStarted
OrderAddressValidated
OrderItemsValidated
OrderPricingValidated
OrderValidationCompleted
OrderPaymentInitiated
OrderPaymentAuthorized
OrderPaymentCaptured
OrderPaymentCompleted
OrderInventoryCheckStarted
OrderInventoryAvailable
OrderInventoryReserved
OrderInventoryCheckCompleted
OrderConfirmed
OrderConfirmationEmailQueued
OrderConfirmationEmailSent
OrderAnalyticsRecorded
```

Eighteen events for one user action. And if the notification service fails and retries, you get more events for the retry. And if the analytics service processes events in batches, it might re-emit its own events for each batch.

**Why it happens:** Over-decomposition. The team has internalized "events are good" and concluded "more events are more good." Or each team is independently logging their internal state transitions as events, not realizing that the aggregate volume is crushing the broker.

**The damage:**
- **Broker overload.** Event volume grows superlinearly with user activity.
- **Consumer lag.** Consumers spend most of their time processing noise events they don't care about.
- **Increased latency.** More events means more broker writes, more network traffic, more consumer processing.
- **Storage costs.** All those events are stored. Retained. Replicated. Backed up.

**How to fix it:**

1. **Distinguish between internal and external events.** `OrderInventoryCheckStarted` is an internal state transition of the `InventoryService`. It should be a log line, not an event on a shared topic.
2. **Publish outcome events, not step events.** `OrderConfirmed` is an outcome. `OrderAddressValidated` is a step. The outside world cares about outcomes.
3. **Batch related state changes.** Instead of 5 events for the payment lifecycle, publish one `PaymentCompleted` event with the relevant details.
4. **Measure your event-to-business-action ratio.** If one user action generates more than 3-5 external events, question whether all of them need to exist.

---

## Premature Event Sourcing — "We Might Need the History Someday"

**The pattern:** The team adopts event sourcing — storing all state as a sequence of events rather than as current state — for every service, regardless of whether the benefits justify the costs. The justification is usually some variation of "it gives us a complete audit trail" or "we might need to replay events someday."

**Why it's a problem:** Event sourcing is a powerful technique with genuine use cases: financial systems where you need a complete audit trail, collaborative editing where you need to merge divergent histories, domains with complex temporal queries. But it's also one of the most operationally expensive architectural patterns in existence.

**The costs nobody mentions in the conference talk:**

- **Event store management.** You now have an append-only log that grows forever. Snapshots help but add complexity. Compaction has different semantics than in a traditional database.
- **Projection rebuilds.** When a read model projection has a bug, you need to replay all events to rebuild it. For a mature system with millions of events, this can take hours or days.
- **Schema evolution is brutal.** Every version of every event format must be deserializable forever. You can't just run a database migration. You need upcasters that convert old event formats to new ones on the fly.
- **Debugging difficulty.** "What's the current state?" requires replaying events. You can't just `SELECT * FROM orders WHERE id = 'ord-123'`.
- **Eventual consistency everywhere.** Read models are projections of the event stream, and they're always at least slightly behind. This is fine for most cases and terrible for others (like showing a user the order they just placed).

```python
# When event sourcing is warranted
class BankAccount:
    """
    Financial regulations require a complete, immutable audit trail.
    Event sourcing is a natural fit.
    """
    def apply(self, event):
        match event:
            case Deposited(amount=amount):
                self.balance += amount
            case Withdrawn(amount=amount):
                self.balance -= amount
            case Frozen(reason=reason):
                self.is_frozen = True

# When event sourcing is NOT warranted
class UserPreferences:
    """
    User changed their notification settings.
    Nobody needs the history of notification preference changes.
    Just use a database.
    """
    pass  # Use a regular UPDATE statement. Seriously.
```

**How to fix it:**

1. **Use event sourcing only where the history is the feature.** If the business requirement is "show me the current state," a database is simpler, faster, and easier to operate.
2. **Event sourcing per aggregate, not per system.** The `BankAccount` aggregate might be event-sourced. The `UserPreferences` aggregate should not be.
3. **CQRS without event sourcing.** You can have separate read and write models (CQRS) without storing the write model as events. Many of the benefits of CQRS come from the separation of concerns, not from the event store.

---

## The Event-Driven Bandwagon — Using EDA Because It's Trendy

**The pattern:** The team adopts event-driven architecture not because the problem demands it, but because it's what the industry is talking about, it looks good on a resume, or someone went to a conference.

**Symptoms:**

- The system has fewer than five services and traffic that a single PostgreSQL database handles comfortably.
- Events are consumed by exactly one consumer (in which case, why not a direct call?).
- The team spent three months setting up Kafka for a system that processes 100 events per day.
- Every architecture discussion includes the phrase "but what if we need to scale?" about a product that has 200 users.

**The honest truth:** Most software systems don't need event-driven architecture. A well-designed monolith with a relational database handles an enormous range of requirements. EDA adds value when you have genuine decoupling requirements, multiple independent consumers for the same data, high throughput demands, or complex workflows that benefit from choreography.

**The test:** Would a synchronous API call between these two services work? Is there a reason it can't be synchronous? If the only reason for using events is "events are better," you don't have a reason.

**How to fix it:** Be honest about your requirements. If you've already deployed the infrastructure, consider whether you can simplify by replacing some event-driven communication with direct calls where appropriate. There's no shame in synchronous communication. It's been powering the internet since before most of your team was born.

---

## Synchronous Disguised as Asynchronous — Request-Reply Over Events

**The pattern:** A service publishes an event and then blocks waiting for a response event. The producer has a correlation ID, a timeout, and a temporary reply topic. Congratulations, you've reinvented HTTP but worse.

```python
# This is a synchronous call pretending to be asynchronous
class OrderService:
    def place_order(self, order):
        correlation_id = str(uuid.uuid4())

        # Publish "request" event
        self.producer.produce("payment-requests", {
            "correlationId": correlation_id,
            "orderId": order.id,
            "amount": order.total,
        })

        # Block waiting for "response" event
        response = self.reply_consumer.wait_for(
            topic="payment-responses",
            correlation_id=correlation_id,
            timeout=30,  # seconds
        )

        if response is None:
            raise TimeoutError("Payment service didn't respond")

        if response["status"] == "APPROVED":
            return OrderConfirmation(order.id)
        else:
            raise PaymentDeclinedException(response["reason"])
```

**Why it's a problem:**
- You've added the latency of two broker hops (request + response) to what could have been one HTTP call.
- You've added the complexity of correlation IDs, reply topics, and timeout handling.
- You've lost the benefits of asynchronous communication (temporal decoupling, independent scaling) because the producer is blocking anyway.
- If the broker is down, the "synchronous" call fails — and you don't even get a clear HTTP error code.

**When request-reply over events IS appropriate:** When the request and response genuinely occur at different times (hours, days), when you need the request to be durably queued, or when you need multiple services to see the request (fan-out with response aggregation). These are rare cases.

**How to fix it:** Use a synchronous call (HTTP, gRPC) for request-response interactions. Use events for fire-and-forget notifications and reactions. If you need the call to be resilient, add retries and a circuit breaker to the HTTP call. That's what those patterns are for.

---

## Schema Anarchy — No Governance, No Contracts, No Hope

**The pattern:** There is no schema registry, no schema validation, and no governance over event formats. Each producer publishes whatever JSON it feels like. Consumers parse with `json.loads()` and hope.

**What it looks like in production:**

```python
# Producer A's idea of an OrderPlaced event
{"type": "order_placed", "order_id": "123", "amount": 59.98}

# Producer B's idea of an OrderPlaced event
{"eventType": "OrderPlaced", "orderId": "ORD-123", "totalAmount": "59.98", "currency": "USD"}

# Producer C's idea (after a Friday afternoon refactor)
{"event": "ORDER_PLACED", "id": "123", "total": 5998}  # amount in cents, because why not

# The consumer
def handle_order(event):
    order_id = event.get("order_id") or event.get("orderId") or event.get("id")
    amount = event.get("amount") or event.get("totalAmount") or event.get("total")
    if isinstance(amount, str):
        amount = float(amount)
    if amount > 1000:  # probably cents?
        amount = amount / 100
    # I hate my life
```

**Why it happens:** Schema governance is unglamorous work. Nobody gets promoted for setting up a schema registry. The team moves fast in the early days, shipping features without formal schemas, and by the time the pain is unbearable, there are 200 event types in production with no consistent format.

**The damage:**
- **Consumer fragility.** Consumers break on every producer change because there's no contract.
- **Silent data corruption.** A producer changes a field from dollars to cents. The consumer doesn't know. Reports are wrong for three weeks.
- **Onboarding difficulty.** New developers cannot understand the system because there's no authoritative documentation of event formats.
- **Impossible schema evolution.** You can't evolve what you haven't defined.

**How to fix it:**

1. **Deploy a schema registry.** Confluent Schema Registry, Apicurio, or even a Git repository with reviewed schema files.
2. **Enforce schema validation on produce.** Events that don't match the registered schema are rejected. No exceptions.
3. **Establish naming conventions.** camelCase or snake_case, pick one. `eventType` or `type`, pick one. Document it. Enforce it in code review.
4. **Require schema review for new event types.** Like database migration review, but for events.

---

## The Dual-Write Problem — Writing to DB and Broker Without Coordination

**The pattern:** A service writes to its database and publishes an event in two separate operations without coordination. If one succeeds and the other fails, the database and the event stream are inconsistent.

```python
# THE BUG
def place_order(self, order):
    # Step 1: Write to database
    self.db.insert(order)        # succeeds

    # Step 2: Publish event
    self.producer.produce(       # FAILS (broker is down)
        "orders",
        OrderPlaced(order)
    )

    # Result: order exists in DB but no event was published.
    # Downstream services don't know the order exists.
    # The customer gets charged but shipping never starts.
```

```python
# THE OTHER BUG (reversing the order doesn't help)
def place_order(self, order):
    # Step 1: Publish event
    self.producer.produce(       # succeeds
        "orders",
        OrderPlaced(order)
    )

    # Step 2: Write to database
    self.db.insert(order)        # FAILS (unique constraint violation)

    # Result: event was published but order doesn't exist in DB.
    # Downstream services try to process a phantom order.
```

**Why it happens:** Developers are accustomed to transactional databases, where two writes either both succeed or both fail. Databases and message brokers are separate systems and don't share transactions (in general).

**How to fix it:**

1. **Transactional outbox pattern.** Write the event to an "outbox" table in the same database transaction as the business data. A separate process reads the outbox and publishes to the broker.

```python
def place_order(self, order):
    with self.db.transaction() as tx:
        tx.insert("orders", order)
        tx.insert("outbox", {
            "topic": "orders",
            "key": order.id,
            "payload": json.dumps(OrderPlaced(order).to_dict()),
            "created_at": datetime.utcnow(),
        })
    # Both writes succeed or both fail — atomic.
    # A separate outbox relay process publishes the events.
```

2. **Change Data Capture (CDC).** Use Debezium or a similar tool to capture changes from the database's transaction log and publish them as events. The database is the source of truth; events are derived.

3. **Event sourcing.** If the event IS the write (event sourcing), there's no dual write — there's only one write.

---

## Missing Idempotency — "It Worked in Dev"

**The pattern:** Consumers process events without any deduplication or idempotency mechanism. In development, with at-most-once delivery and a single consumer, everything looks fine. In production, with at-least-once delivery, rebalancing, and retries, customers get charged twice.

```python
# NOT IDEMPOTENT — will charge the customer for every delivery attempt
class PaymentConsumer:
    def handle(self, event):
        if event["eventType"] == "OrderPlaced":
            self.payment_gateway.charge(
                customer_id=event["data"]["customerId"],
                amount=event["data"]["totalAmount"],
            )
            # If the consumer crashes AFTER charging but BEFORE committing
            # the offset, the event will be redelivered and the customer
            # will be charged again.
```

**Why it happens:** At-least-once delivery semantics mean that under normal operation, most events are delivered exactly once. The duplicates appear during edge cases: consumer rebalancing, broker failover, network hiccups, process crashes. These don't happen in local development. They happen at 3 AM on Saturday in production.

**How to fix it:**

```python
# IDEMPOTENT — safe to process multiple times
class PaymentConsumer:
    def handle(self, event):
        if event["eventType"] == "OrderPlaced":
            event_id = event["eventId"]

            # Check if we've already processed this event
            if self.dedup_store.has_been_processed(event_id):
                logger.info(f"Skipping duplicate event {event_id}")
                return

            # Process the event
            self.payment_gateway.charge(
                customer_id=event["data"]["customerId"],
                amount=event["data"]["totalAmount"],
                idempotency_key=event_id,  # payment gateway also deduplicates
            )

            # Record that we've processed this event
            self.dedup_store.mark_processed(event_id)
```

Better yet, use natural idempotency keys. Instead of a generic `eventId`, use a domain-specific key that makes duplicates naturally harmless:

```sql
-- Idempotent via unique constraint
INSERT INTO payments (order_id, amount, status)
VALUES ('ord-123', 59.98, 'CHARGED')
ON CONFLICT (order_id) DO NOTHING;
-- Second insert is silently ignored. No duplicate charge.
```

---

## Ignoring Back-Pressure — The Firehose Problem

**The pattern:** A producer publishes events at a rate far exceeding what consumers can process. There's no mechanism to slow the producer down, no monitoring of consumer lag, and no alerting until the broker's disk fills up.

**What it looks like:**

```
Producer: 50,000 events/sec
Consumer: 5,000 events/sec
Lag growth: 45,000 events/sec
Time to fill broker disk: ~4 hours
Time until alert fires: never (nobody set one up)
Time until on-call page: when the broker crashes
```

**Why it happens:** The producer and consumer are developed by different teams. The producer team load-tested their producer. The consumer team load-tested their consumer. Nobody tested them together at production-grade volumes.

**How to fix it:**

1. **Monitor consumer lag.** This is the single most important metric for any event-driven system. Alert when lag exceeds a threshold.
2. **Set broker-side quotas.** Limit per-producer and per-consumer throughput.

```properties
# Kafka producer quotas
quota.producer.default=10485760  # 10 MB/sec per producer
quota.consumer.default=10485760  # 10 MB/sec per consumer
```

3. **Right-size your consumer parallelism.** If a single consumer can handle 5,000 events/sec and you're producing 50,000 events/sec, you need at least 10 consumer instances (and at least 10 partitions).
4. **Implement backpressure in the producer** when possible. If the producer is ingesting from an external source, it may be able to slow down or buffer when the downstream system is overwhelmed.
5. **Set retention limits** that match your consumer's ability to catch up. If your consumer can never process a month of events in a reasonable time, a month-long retention policy gives you false confidence.

---

## Over-Partitioning and Under-Partitioning

### Over-Partitioning

**The pattern:** "We might need to scale to 100 consumers someday, so let's create 100 partitions now." The system has 3 consumers and 100 partitions. 97 partitions sit idle. Broker metadata overhead increases. Rebalancing takes longer. Leader election after a broker failure is slower.

**The costs of too many partitions:**
- Each partition has a leader and replicas. More partitions = more metadata, more leader election overhead.
- Consumer rebalancing time increases linearly with partition count.
- End-to-end latency increases because the broker batches by partition, and more partitions mean smaller batches.
- File descriptor usage on the broker increases (each partition has at least one open segment file per replica).

### Under-Partitioning

**The pattern:** The topic has 1 partition. The consumer cannot be parallelized. When load increases, the only option is to process faster — you cannot add more consumers.

**The costs of too few partitions:**
- Maximum consumer parallelism equals the partition count. One partition = one consumer.
- You can increase the partition count later, but you can't decrease it (in Kafka). And increasing it breaks key-based ordering guarantees for existing keys.

**How to fix it:** Start with a partition count based on your expected peak throughput and consumer parallelism, with modest headroom. A common heuristic: `max(throughput_mbps / consumer_throughput_per_partition_mbps, expected_max_consumers)`. For most workloads, 6-30 partitions per topic is reasonable. 100+ is almost always premature. 1 is almost always insufficient.

---

## The "Just Replay Everything" Fallacy

**The pattern:** "If anything goes wrong with a consumer's state, we'll just replay all events from the beginning." This sounds reasonable when you have 10,000 events. It stops sounding reasonable when you have 10 billion.

**The problems:**

- **Replay time.** Replaying a year of events for a single consumer can take days. The consumer is unavailable during replay, or serving stale data.
- **Side effects.** If the consumer's event handler has side effects (sending emails, charging credit cards, calling external APIs), replaying events re-triggers those side effects. You now need to distinguish between "live" processing and "replay" processing, which adds complexity to every handler.
- **Schema evolution.** Events from a year ago might be in a format that the current consumer doesn't support. You need event upcasters or versioned handlers.
- **Resource consumption.** Replaying generates enormous read load on the broker. If other consumers are sharing the same broker, their performance degrades.

**How to fix it:**

1. **Take periodic snapshots.** Instead of replaying from the beginning, replay from the last known good snapshot. This bounds the replay window.
2. **Build idempotent consumers.** If replay is a recovery mechanism, the consumer must handle replayed events safely (see "Missing Idempotency" above).
3. **Design handlers to detect replay mode.** Suppress side effects during replay (no emails, no API calls, no charges).
4. **Set realistic retention policies.** If you can't replay more than a week's worth of events in a reasonable time, a 90-day retention policy is giving you 83 days of false comfort.
5. **Monitor replay progress.** If you're replaying, know how long it will take. "It's replaying" is not a status. "It's replayed 4.2 billion of 7.8 billion events, estimated completion in 9 hours" is a status.

---

## How to Recognize You're in Trouble and How to Dig Out

### Warning Signs

You might already be living with some of these anti-patterns. Here's how to tell:

**Symptoms of Event Soup:**
- Your event catalog has more than 100 event types and nobody can explain what half of them are for.
- New team members take more than a week to understand the event flow.
- You have topics with no active consumers.

**Symptoms of a Distributed Monolith:**
- Deploying one service requires coordinating with three other teams.
- A failure in one service cascades to multiple downstream services within seconds.
- Your deployment pipeline has a specific service ordering.

**Symptoms of Missing Idempotency:**
- Customers report duplicate charges, duplicate notifications, or duplicate orders "sometimes."
- The bugs are never reproducible in development.
- Your on-call rotation coincides with broker maintenance windows.

**Symptoms of Schema Anarchy:**
- Consumer code is full of `try/except` blocks around deserialization.
- Field names change without warning.
- Nobody knows the authoritative format for any event type.

**Symptoms of the Firehose Problem:**
- Consumer lag grows during business hours and shrinks overnight.
- Broker disk usage grows monotonically.
- End-to-end latency increases throughout the day.

### Digging Out

If you recognize these symptoms, here's the uncomfortable truth: fixing anti-patterns in a running system is harder than preventing them. But it's not impossible.

**Step 1: Observe.** Before changing anything, instrument the system. Add consumer lag monitoring. Build an event flow diagram. Catalog every event type and its producers/consumers. You can't fix what you can't see.

**Step 2: Prioritize by blast radius.** The dual-write problem that occasionally loses orders is more urgent than the chatty service that wastes bandwidth. Fix the things that cause data loss or incorrect behavior first.

**Step 3: Introduce governance incrementally.** You don't need to boil the ocean. Start with a schema registry and require schemas for new event types. Existing unschematized events can be migrated gradually.

**Step 4: Fix idempotency.** This is the single highest-value improvement for most event-driven systems. Make every consumer idempotent. Use the outbox pattern for producers. This doesn't fix the architecture, but it prevents the architecture's problems from reaching customers.

**Step 5: Consolidate event types.** Kill the event types nobody consumes. Merge the event types that differ by one field. Replace the god events with focused events. This is slow, thankless work, and it's the most impactful architectural improvement you can make.

**Step 6: Establish ownership.** Every topic has an owning team. Every event type has an owning producer. Every schema has a reviewer. Without ownership, entropy wins.

---

## Summary

Event-driven architecture is not inherently better or worse than other architectural styles. It's a set of trade-offs. The anti-patterns in this chapter all share a common root cause: adopting the style without fully understanding the trade-offs, or understanding them in theory but not in the operational reality of a production system.

The good news: every anti-pattern here has been encountered, diagnosed, and survived by teams before you. The bad news: many of those teams encountered it in production, diagnosed it under pressure, and survived it by the narrowest of margins.

Read this chapter before you build. Reread it six months after you ship. The anti-patterns you recognize the second time will be different — and more personally relevant — than the ones you recognized the first time.
