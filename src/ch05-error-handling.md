# Error Handling and Delivery Guarantees

In a synchronous system, error handling is straightforward: the call fails, you get an exception, you show the user a sad face, and everyone moves on. In an event-driven system, error handling is a philosophy, a lifestyle, and occasionally a source of existential dread.

The fundamental challenge is this: when a producer publishes an event and walks away, who is responsible when something goes wrong? The producer has already moved on. The broker is just a pipe. The consumer might not even exist yet. The event could fail to process three days from now, on a server in a region you've never heard of, for a reason that has nothing to do with the original business logic. And you need a plan for that.

This chapter covers the guarantees your system can (and cannot) provide, the strategies for handling failures gracefully, and the tools for dealing with events that refuse to be processed.

---

## The Three Delivery Guarantees (and What They Actually Mean)

Every messaging system documentation page features a section on delivery guarantees, typically presented with the gravitas of constitutional law. There are three:

### At-Most-Once

The event is delivered zero or one times. It might be lost, but it will never be duplicated.

Implementation: the producer fires the event and doesn't wait for acknowledgment. Or the broker acknowledges receipt but the consumer doesn't acknowledge processing. If anything goes wrong — network blip, consumer crash, broker hiccup — the event is gone.

```python
# At-most-once: fire and forget
producer.send('orders', event)
# Did it arrive? Who knows. Moving on.
```

**When it's appropriate:** Metrics, analytics, logging — data where losing a few events is acceptable and duplicates would skew your numbers. If your dashboard can tolerate showing 99.97% of events instead of 100%, at-most-once is simpler and faster.

**When it's not:** Financial transactions, order processing, anything where losing an event means losing money or trust.

### At-Least-Once

The event is delivered one or more times. It will never be lost, but it might be duplicated.

Implementation: the producer retries until it gets an acknowledgment. The consumer processes the event and then acknowledges it. If the consumer crashes after processing but before acknowledging, the broker redelivers the event, and the consumer processes it again.

```python
# At-least-once: producer with retries
producer.send('orders', event, acks='all', retries=3)

# Consumer side: process then commit
event = consumer.poll()
process(event)         # This succeeds
consumer.commit()      # But what if we crash here?
# If we crash between process() and commit(),
# the event will be redelivered. We'll process it twice.
```

This is the most common guarantee in practice, because it's achievable without exotic infrastructure. The tradeoff is that your consumers must handle duplicates, which is the topic of the next section.

### Exactly-Once

The event is delivered exactly one time. Never lost, never duplicated. The holy grail.

And now for the uncomfortable part.

---

## Why "Exactly-Once" Is Mostly Marketing

"Exactly-once delivery" is one of those phrases that sounds simple, means something specific and narrow in the contexts where it's achievable, and means something impossible in the general case. Let's untangle it.

### The Two Generals Problem

Distributed systems theory has proven — *proven*, mathematically, not just "it's really hard" — that exactly-once delivery between two independent processes over an unreliable network is impossible. This is a consequence of the Two Generals Problem. You cannot guarantee that both the sender and receiver agree on whether a message was delivered, because the acknowledgment itself can be lost.

If the producer sends an event and the ack gets lost, the producer doesn't know if the event was received. It can either:
- **Not retry:** risking event loss (at-most-once).
- **Retry:** risking duplication (at-least-once).

There is no third option. Physics doesn't care about your SLA.

### What "Exactly-Once" Actually Means in Practice

When Kafka or Pulsar or any other system claims "exactly-once," they mean one of two things:

1. **Exactly-once within the broker.** Kafka's exactly-once semantics (EOS) guarantee that a produce-consume-produce cycle within Kafka doesn't duplicate events. The broker uses producer IDs and sequence numbers to deduplicate, and transactions to ensure atomic writes across multiple partitions. This is real, it works, and it's a significant engineering achievement. But it only applies to the Kafka-to-Kafka pipeline.

2. **Effectively-once with idempotent consumers.** The system delivers at-least-once, but the consumer is designed so that processing the same event multiple times has the same effect as processing it once. The event might be *delivered* more than once, but it's *processed* exactly once in terms of its effect on the world.

```java
// Kafka exactly-once: transactional produce-consume-produce
Properties props = new Properties();
props.put("transactional.id", "order-processor-1");
props.put("enable.idempotence", true);

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
producer.initTransactions();

try {
    producer.beginTransaction();

    // Consume from input topic
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, String> record : records) {
        // Process and produce to output topic
        ProducerRecord<String, String> output = transform(record);
        producer.send(output);
    }

    // Atomically commit consumer offsets and producer writes
    producer.sendOffsetsToTransaction(
        currentOffsets, consumerGroupMetadata);
    producer.commitTransaction();

} catch (Exception e) {
    producer.abortTransaction();
}
```

This is powerful and useful. It's also limited to the Kafka-internal pipeline. The moment your consumer calls an external HTTP API, writes to a database, or sends an email, you're back to at-least-once with idempotency requirements.

### The Honest Version

Here's what you should tell your stakeholders: "Our system provides at-least-once delivery with idempotent processing, which means every event will be processed, and processing it more than once won't cause incorrect behavior. This is what the industry calls 'effectively once' and it's what 'exactly-once' actually means in any real-world system that interacts with external services."

That's less catchy than "exactly-once," but it has the advantage of being true.

---

## Idempotent Consumers: The Real Solution

Since you're going to receive duplicates — and you are, it's not a question of if — your consumers need to handle them gracefully. An idempotent operation is one where performing it multiple times has the same result as performing it once.

### Strategies for Idempotency

#### 1. Natural Idempotency

Some operations are naturally idempotent:
- `SET balance = 100` (idempotent: same result every time)
- `INSERT OR UPDATE customer SET name = 'Jane'` (idempotent)
- `DELETE FROM orders WHERE id = 'ord-123'` (idempotent after first execution)

Some are not:
- `INCREMENT balance BY 10` (not idempotent: each execution adds 10)
- `INSERT INTO ledger (amount) VALUES (10)` (not idempotent: creates new rows)
- `SEND EMAIL to customer` (very not idempotent: customer gets annoyed)

#### 2. Deduplication with Event IDs

Every event should carry a unique identifier. The consumer tracks which IDs it has already processed and skips duplicates.

```python
class IdempotentConsumer:
    def __init__(self, db):
        self.db = db

    def handle_event(self, event):
        event_id = event['metadata']['eventId']

        # Check if we've already processed this event
        if self.db.exists('processed_events', event_id):
            log.info(f"Skipping duplicate event: {event_id}")
            return

        # Process the event within a transaction
        with self.db.transaction() as tx:
            self._process(event, tx)

            # Record that we've processed this event
            tx.insert('processed_events', {
                'event_id': event_id,
                'processed_at': datetime.utcnow(),
                'consumer': self.consumer_name
            })

    def _process(self, event, tx):
        # Actual business logic here
        order = event['payload']
        tx.insert('orders', {
            'id': order['orderId'],
            'customer_id': order['customerId'],
            'total': order['totalAmount']
        })
```

**Critical detail:** The business logic and the deduplication record must be in the same transaction. If you process the event, crash before recording it, and then process it again on redelivery, you've defeated the purpose.

#### 3. Idempotency Keys in External Calls

When your consumer calls an external service (payment gateway, email provider, shipping API), pass an idempotency key so the external service can deduplicate on its end.

```python
def process_payment(event):
    idempotency_key = f"payment-{event['orderId']}-{event['eventId']}"

    response = payment_gateway.charge(
        amount=event['totalAmount'],
        currency=event['currency'],
        idempotency_key=idempotency_key  # Gateway deduplicates on this
    )

    return response
```

Most serious payment APIs support this. Stripe, for example, accepts an `Idempotency-Key` header that guarantees the same charge isn't processed twice, regardless of how many times you call the API with that key.

#### 4. Conditional Writes

Use optimistic concurrency control to make writes idempotent:

```python
def apply_discount(event, db):
    order_id = event['orderId']
    expected_version = event['orderVersion']

    rows_affected = db.execute("""
        UPDATE orders
        SET discount = %s, version = version + 1
        WHERE id = %s AND version = %s
    """, [event['discount'], order_id, expected_version])

    if rows_affected == 0:
        # Either the order doesn't exist or the version doesn't match.
        # If the version doesn't match, we've already applied this
        # (or a later) update. Either way, we're done.
        log.info(f"Conditional write skipped for order {order_id}")
```

---

## Retry Strategies

When processing fails, you retry. But *how* you retry matters enormously. A naive retry strategy can turn a transient failure into a cascading outage.

### Immediate Retry

Retry instantly. This works for genuinely transient errors — a momentary network blip, a brief connection pool exhaustion. It fails catastrophically for errors that need time to resolve, because you're hammering the failing service at full speed.

```python
# Don't do this in production without a limit
def process_with_immediate_retry(event, max_retries=3):
    for attempt in range(max_retries):
        try:
            return process(event)
        except TransientError:
            if attempt == max_retries - 1:
                raise
            continue  # Try again immediately
```

**Use when:** The error is almost certainly a momentary glitch, and the downstream service can handle the retry volume. So, almost never.

### Fixed Delay

Wait a fixed amount of time between retries. Better than immediate retry because it gives the downstream system time to recover.

```python
def process_with_fixed_delay(event, max_retries=3, delay_seconds=5):
    for attempt in range(max_retries):
        try:
            return process(event)
        except TransientError:
            if attempt == max_retries - 1:
                raise
            time.sleep(delay_seconds)
```

**Use when:** You have a rough idea of how long recovery takes and the volume of retries is modest.

### Exponential Backoff

Each retry waits longer than the last: 1s, 2s, 4s, 8s, 16s, and so on. This is the standard approach for most failure scenarios, because it starts optimistic (maybe it's a quick fix) and becomes progressively more patient.

```python
def process_with_exponential_backoff(event, max_retries=5, base_delay=1):
    for attempt in range(max_retries):
        try:
            return process(event)
        except TransientError:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt)  # 1, 2, 4, 8, 16
            time.sleep(delay)
```

### Exponential Backoff with Jitter

Here's the problem with exponential backoff: if 100 consumers all fail at the same time (because a downstream service went down), they all retry at the same time, creating a thundering herd that takes down the service again as soon as it recovers.

Jitter randomizes the retry delay to spread out the load:

```python
import random

def process_with_backoff_and_jitter(event, max_retries=5, base_delay=1, max_delay=60):
    for attempt in range(max_retries):
        try:
            return process(event)
        except TransientError:
            if attempt == max_retries - 1:
                raise

            # Full jitter: random delay between 0 and the exponential ceiling
            exp_delay = min(base_delay * (2 ** attempt), max_delay)
            delay = random.uniform(0, exp_delay)
            time.sleep(delay)
```

AWS's architecture blog has an excellent analysis of jitter strategies. The "full jitter" approach (random between 0 and the exponential ceiling) outperforms both "equal jitter" and "decorrelated jitter" in most scenarios. Use full jitter. Your future self will thank you.

### The Retry Budget

Beyond per-event retries, consider a system-wide retry budget: a limit on the total number of retries per time window across all events.

```python
class RetryBudget:
    def __init__(self, max_retries_per_minute=100):
        self.max_retries = max_retries_per_minute
        self.retry_count = 0
        self.window_start = time.time()

    def can_retry(self):
        now = time.time()
        if now - self.window_start > 60:
            self.retry_count = 0
            self.window_start = now

        return self.retry_count < self.max_retries

    def record_retry(self):
        self.retry_count += 1

retry_budget = RetryBudget(max_retries_per_minute=100)

def process_with_budget(event):
    try:
        return process(event)
    except TransientError:
        if retry_budget.can_retry():
            retry_budget.record_retry()
            requeue(event)
        else:
            send_to_dlq(event)
```

Without a retry budget, a sustained downstream outage can cause your retry queue to grow without bound, consuming memory and network resources and potentially causing your consumer to fall behind on non-failing events.

---

## Dead Letter Queues: Your Event Purgatory

When an event has exhausted its retries and still can't be processed, it goes to the dead letter queue (DLQ). The DLQ is where events go to wait for a human to figure out what went wrong.

### Anatomy of a Good DLQ

A DLQ isn't just a dumping ground. A well-designed DLQ includes:

1. **The original event**, unmodified.
2. **Error metadata**: the exception message, stack trace, consumer name, timestamp of last failure, number of retry attempts.
3. **Routing metadata**: which topic it came from, which consumer group failed on it, which partition and offset.

```python
def send_to_dlq(event, error, context):
    dlq_envelope = {
        'originalEvent': event,
        'error': {
            'message': str(error),
            'type': type(error).__name__,
            'stackTrace': traceback.format_exc(),
            'timestamp': datetime.utcnow().isoformat()
        },
        'source': {
            'topic': context.topic,
            'partition': context.partition,
            'offset': context.offset,
            'consumerGroup': context.consumer_group
        },
        'retryHistory': {
            'attempts': context.retry_count,
            'firstAttempt': context.first_attempt_time.isoformat(),
            'lastAttempt': datetime.utcnow().isoformat()
        }
    }

    producer.send(f"{context.topic}.dlq", dlq_envelope)
```

### DLQ Processing Patterns

Events in the DLQ aren't dead — they're in purgatory. You need processes for dealing with them:

**Manual review and replay:** An operator examines the failed event, fixes the underlying issue (deploys a bug fix, corrects bad data), and replays the event back to the original topic.

**Automated retry with delay:** A separate consumer reads from the DLQ, waits a configurable period (hours or days, not seconds), and resubmits events to the original topic. This handles cases where the failure was caused by a temporary condition that resolved itself.

**Automated triage:** A DLQ consumer classifies errors and routes events to different handling queues:
- Deserialization errors → schema mismatch queue (probably needs a code fix)
- Timeout errors → delayed retry queue (probably transient)
- Validation errors → data quality queue (probably needs manual correction)

```python
class DLQTriageConsumer:
    def handle(self, dlq_event):
        error_type = dlq_event['error']['type']

        if error_type in ('SerializationError', 'SchemaError'):
            self.route_to('schema-issues', dlq_event)
        elif error_type in ('TimeoutError', 'ConnectionError'):
            self.route_to('delayed-retry', dlq_event)
        elif error_type == 'ValidationError':
            self.route_to('data-quality', dlq_event)
        else:
            self.route_to('unknown-errors', dlq_event)
            self.alert_on_call_engineer(dlq_event)
```

### The DLQ Naming Convention

Use a consistent naming scheme so it's obvious which DLQ belongs to which topic:

```
orders.order-created           → orders.order-created.dlq
payments.payment-processed     → payments.payment-processed.dlq
```

Or, for consumer-specific DLQs (when multiple consumers process the same topic and might fail for different reasons):

```
orders.order-created.fulfillment-service.dlq
orders.order-created.analytics-service.dlq
```

---

## Poison Pills: Events That Will Never Succeed

A poison pill is an event that will cause the consumer to fail no matter how many times you retry it. Corrupted data, malformed JSON, an event that triggers a bug in your processing logic — these will never succeed, and retrying them is worse than useless.

### Identifying Poison Pills

The first step is recognizing that an event is a poison pill rather than a transient failure:

```python
class EventProcessor:
    MAX_RETRIES_TRANSIENT = 5
    MAX_RETRIES_TOTAL = 10

    def process(self, event, retry_count):
        try:
            # Deserialization — if this fails, it's a poison pill
            parsed = self.deserialize(event)
        except (json.JSONDecodeError, SchemaError) as e:
            # Deterministic failure. Don't retry.
            self.send_to_dlq(event, e, poison_pill=True)
            return

        try:
            # Business logic — might be transient or permanent
            self.handle(parsed)
        except TransientError:
            if retry_count < self.MAX_RETRIES_TRANSIENT:
                self.retry(event, retry_count + 1)
            else:
                self.send_to_dlq(event, e, poison_pill=False)
        except ValidationError as e:
            # Deterministic business logic failure
            self.send_to_dlq(event, e, poison_pill=True)
        except Exception as e:
            # Unknown error — retry a few times, then DLQ
            if retry_count < self.MAX_RETRIES_TOTAL:
                self.retry(event, retry_count + 1)
            else:
                self.send_to_dlq(event, e, poison_pill=False)
```

The key insight: **categorize errors as deterministic (poison pill) or non-deterministic (transient)**. Deterministic failures should go straight to the DLQ — retrying them wastes time and, more importantly, blocks processing of subsequent events if you're consuming from an ordered partition.

### The Poison Pill Partition Problem

In Kafka, messages within a partition are processed in order. If your consumer encounters a poison pill and keeps retrying it, every subsequent message in that partition is blocked. This is the single most common cause of "consumer lag alert firing, nobody knows why."

```
Partition 0: [msg1] [msg2] [POISON] [msg4] [msg5] [msg6] ...
                                ↑
                     Consumer stuck here.
                     msg4, msg5, msg6 are waiting.
                     Lag is growing.
                     Your pager is about to go off.
```

The solution: detect poison pills quickly (within 1-2 retries), send them to the DLQ, and move on. Do not allow a single bad event to block an entire partition.

---

## Circuit Breakers in Async Systems

The circuit breaker pattern, borrowed from electrical engineering, prevents a failing downstream service from being hammered with requests. In synchronous systems, it's well-understood: after N consecutive failures, the circuit "opens" and all requests fail fast without calling the downstream service. After a timeout, the circuit enters "half-open" state and allows a single test request through.

In async systems, circuit breakers are trickier because the consumer doesn't make synchronous calls in the traditional sense. But the principle still applies when your consumer depends on external services.

```python
class CircuitBreaker:
    CLOSED = 'closed'
    OPEN = 'open'
    HALF_OPEN = 'half_open'

    def __init__(self, failure_threshold=5, reset_timeout=60):
        self.state = self.CLOSED
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.reset_timeout = reset_timeout
        self.last_failure_time = None

    def call(self, func, *args, **kwargs):
        if self.state == self.OPEN:
            if time.time() - self.last_failure_time > self.reset_timeout:
                self.state = self.HALF_OPEN
            else:
                raise CircuitOpenError("Circuit breaker is open")

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        self.failure_count = 0
        self.state = self.CLOSED

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = self.OPEN


# Usage in an event consumer
circuit_breaker = CircuitBreaker(failure_threshold=5, reset_timeout=30)

def process_order(event):
    try:
        circuit_breaker.call(payment_service.charge, event['orderId'], event['amount'])
    except CircuitOpenError:
        # Don't even try — the payment service is down.
        # Pause consumption or route to a retry topic.
        requeue_with_delay(event, delay_seconds=30)
```

### The Consumer Pause Pattern

When the circuit opens, you have a choice: let events pile up in the broker (consumer lag grows) or pause the consumer. Pausing is usually better:

```python
class CircuitAwareConsumer:
    def run(self):
        while True:
            if self.circuit_breaker.state == CircuitBreaker.OPEN:
                # Stop fetching new events until the circuit closes
                log.warning("Circuit open — pausing consumer")
                time.sleep(self.circuit_breaker.reset_timeout)
                continue

            events = self.consumer.poll(timeout_ms=1000)
            for event in events:
                self.process(event)
```

This keeps the events safe in the broker (where they have retention and replay capability) rather than accumulating in memory.

---

## Ordering vs. Retry Tension

Here's a genuinely thorny problem: what happens when you need both ordered processing and retry capability?

### The Problem

Consider a partition with events for the same entity:

```
Partition 0: [OrderCreated:ord-1] [OrderUpdated:ord-1] [OrderShipped:ord-1]
```

These must be processed in order — you can't ship an order before creating it. But what if `OrderCreated` fails transiently? You want to retry it, but you can't skip ahead to `OrderUpdated` while you wait.

If you retry by requeuing to the end of the topic, you've now got:

```
Partition 0: ... [OrderUpdated:ord-1] [OrderShipped:ord-1] ... [OrderCreated:ord-1]
```

Processing `OrderUpdated` before `OrderCreated` is incorrect. Processing `OrderShipped` before either is nonsensical.

### Solutions

#### 1. Block and Retry In-Place

The simplest approach: keep retrying the failing event without advancing the consumer offset. Subsequent events in the partition wait.

```python
def consume_with_ordered_retry(consumer, max_retries=10):
    while True:
        event = consumer.poll()
        retries = 0
        while retries < max_retries:
            try:
                process(event)
                consumer.commit()
                break
            except TransientError:
                retries += 1
                time.sleep(exponential_backoff(retries))
        else:
            # Exhausted retries — DLQ and move on
            send_to_dlq(event)
            consumer.commit()
```

**Downside:** You're blocking the entire partition. If the failure persists, lag grows, and every entity in that partition is affected, not just the one with the failing event.

#### 2. Per-Entity Retry with Buffering

Track which entities are "in retry" and buffer subsequent events for those entities while processing events for other entities normally.

```python
class OrderedRetryConsumer:
    def __init__(self):
        self.retry_buffer = defaultdict(list)  # entity_id -> [events]
        self.entities_in_retry = set()

    def process(self, event):
        entity_id = event['entityId']

        if entity_id in self.entities_in_retry:
            # Buffer this event — we'll process it after the retry succeeds
            self.retry_buffer[entity_id].append(event)
            return

        try:
            handle(event)
        except TransientError:
            self.entities_in_retry.add(entity_id)
            schedule_retry(event, on_success=self._flush_buffer,
                          on_failure=self._send_entity_to_dlq)

    def _flush_buffer(self, entity_id):
        self.entities_in_retry.discard(entity_id)
        for buffered_event in self.retry_buffer.pop(entity_id, []):
            self.process(buffered_event)

    def _send_entity_to_dlq(self, entity_id):
        self.entities_in_retry.discard(entity_id)
        for buffered_event in self.retry_buffer.pop(entity_id, []):
            send_to_dlq(buffered_event, reason="Prior event for entity failed")
```

**Downside:** Complexity. Memory pressure if many entities are in retry simultaneously. You're reimplementing a significant chunk of what the broker does.

#### 3. Retry Topics with Ordering Keys

Use a dedicated retry topic with the same partitioning key, so entity ordering is maintained within the retry flow:

```
Main topic (partition by orderId): events flow normally
  ↓ (on failure)
Retry topic (partition by orderId): failed events, with delay
  ↓ (after delay)
Main topic: retried events rejoin the main flow
```

This is the approach Uber uses in their event processing infrastructure (documented in their engineering blog), and it's well-suited for high-volume systems.

---

## Transactional Outbox: Reliable Publishing

There's a failure mode that trips up every event-driven system eventually: the dual-write problem. Your service needs to update its database AND publish an event, atomically. If either succeeds without the other, the system is inconsistent.

```python
# THE DANGEROUS WAY — dual write
def create_order(order):
    db.insert('orders', order)       # Step 1: write to DB
    producer.send('orders', event)   # Step 2: publish event
    # What if we crash between step 1 and step 2?
    # DB has the order, but no event was published.
    # Or: what if step 1 succeeds, step 2 fails?
```

### The Outbox Pattern

Instead of publishing directly, write the event to an "outbox" table in the same database transaction as the business data. A separate process reads the outbox and publishes to the broker.

```python
def create_order(order):
    with db.transaction() as tx:
        # Business data and event in the SAME transaction
        tx.insert('orders', order.to_dict())
        tx.insert('outbox', {
            'id': uuid4(),
            'aggregate_type': 'Order',
            'aggregate_id': order.id,
            'event_type': 'OrderCreated',
            'payload': json.dumps(order.to_event()),
            'created_at': datetime.utcnow(),
            'published': False
        })
    # Transaction commits atomically — both or neither.
```

The outbox publisher runs as a separate process:

```python
class OutboxPublisher:
    def __init__(self, db, producer, poll_interval=1):
        self.db = db
        self.producer = producer
        self.poll_interval = poll_interval

    def run(self):
        while True:
            events = self.db.query(
                "SELECT * FROM outbox WHERE published = FALSE "
                "ORDER BY created_at LIMIT 100"
            )

            for event in events:
                try:
                    self.producer.send(
                        topic=f"{event['aggregate_type'].lower()}s",
                        key=event['aggregate_id'],
                        value=event['payload']
                    )
                    self.db.update('outbox',
                        {'published': True, 'published_at': datetime.utcnow()},
                        where={'id': event['id']})
                except Exception as e:
                    log.error(f"Failed to publish outbox event {event['id']}: {e}")
                    # Will retry on next poll

            time.sleep(self.poll_interval)
```

### Change Data Capture as an Alternative

Instead of polling the outbox table, use change data capture (CDC) to stream changes from the outbox table to the broker. Debezium is the standard tool for this:

```
[Application] → writes to → [Database (outbox table)]
                                    ↓ CDC
                              [Debezium Connector]
                                    ↓
                              [Kafka Topic]
```

CDC eliminates the polling overhead and provides lower latency. It also means your application doesn't need to know about the broker at all — it just writes to the database, and the infrastructure handles the rest.

**The tradeoff:** CDC adds operational complexity (you're running Debezium, which needs its own monitoring and care). But for high-volume systems, it's worth it.

---

## Error Handling in Sagas: Compensating Transactions

A saga is a sequence of local transactions across multiple services, where each step either succeeds or triggers compensating actions to undo the effects of previous steps. Error handling in sagas is where things get genuinely interesting — and by "interesting" I mean "complex enough to warrant its own whiteboard session."

### The Choreography Approach

In a choreographed saga, each service listens for events and decides what to do next, including how to compensate:

```
1. OrderService: OrderCreated →
2. PaymentService: (hears OrderCreated) → PaymentCharged →
3. InventoryService: (hears PaymentCharged) → InventoryReserved →
4. ShippingService: (hears InventoryReserved) → ShipmentScheduled

# But what if step 3 fails?
3. InventoryService: (hears PaymentCharged) → InventoryReservationFailed →
2. PaymentService: (hears InventoryReservationFailed) → PaymentRefunded →
1. OrderService: (hears PaymentRefunded) → OrderCancelled
```

Every forward step has a corresponding compensating step. The compensating steps must be idempotent (because they might be triggered more than once) and must be tolerant of partial state (because the forward step might have partially completed).

### The Orchestration Approach

An orchestrator service coordinates the saga and handles failures explicitly:

```python
class OrderSaga:
    def __init__(self, order_id):
        self.order_id = order_id
        self.state = 'STARTED'
        self.completed_steps = []

    def execute(self):
        steps = [
            ('reserve_inventory', self.reserve_inventory, self.release_inventory),
            ('charge_payment', self.charge_payment, self.refund_payment),
            ('schedule_shipping', self.schedule_shipping, self.cancel_shipping),
        ]

        for step_name, forward, compensate in steps:
            try:
                forward()
                self.completed_steps.append((step_name, compensate))
                self.state = f'{step_name}_COMPLETED'
            except Exception as e:
                log.error(f"Saga step {step_name} failed: {e}")
                self.state = f'{step_name}_FAILED'
                self._compensate()
                return

        self.state = 'COMPLETED'

    def _compensate(self):
        # Compensate in reverse order
        for step_name, compensate in reversed(self.completed_steps):
            try:
                compensate()
            except Exception as e:
                # Compensation failure — this is the nightmare scenario.
                # Log it, alert a human, and pray.
                log.critical(
                    f"COMPENSATION FAILED for {step_name}: {e}. "
                    f"Manual intervention required for order {self.order_id}"
                )
```

### When Compensation Fails

What happens when the compensating transaction itself fails? This is the question that keeps saga designers up at night.

Options:
1. **Retry the compensation** with exponential backoff. Most compensation failures are transient.
2. **Log and alert.** A human investigates and manually corrects the state.
3. **Compensation journal.** Write all pending compensations to a durable store. A background process retries them until they succeed.

There is no fully automatic solution. At some point, a human may need to reconcile state across services. Design your system so that identifying and fixing inconsistencies is possible, even if it's not pleasant.

---

## Monitoring and Alerting on DLQs

A DLQ that nobody monitors is worse than no DLQ at all — it gives you the illusion of safety while events silently rot.

### Essential DLQ Metrics

```yaml
# Prometheus-style metrics you should be tracking
dlq_events_total:
  description: "Total events sent to DLQ"
  labels: [source_topic, consumer_group, error_type]
  alert: "Rate > 10/min for 5 minutes"

dlq_events_pending:
  description: "Events in DLQ not yet resolved"
  labels: [source_topic, consumer_group]
  alert: "Count > 100 for 30 minutes"

dlq_event_age_seconds:
  description: "Age of oldest unresolved DLQ event"
  labels: [source_topic, consumer_group]
  alert: "Age > 3600 (1 hour)"

dlq_resolution_rate:
  description: "Rate of events being resolved from DLQ"
  labels: [source_topic, resolution_type]  # replay, discard, manual_fix
```

### The DLQ Dashboard

Your DLQ dashboard should answer these questions at a glance:

1. **How many events are in purgatory right now?** (Total, by source topic, by error type.)
2. **Is the inflow rate increasing?** (A spike means something broke. A gradual increase means something is degrading.)
3. **How old is the oldest event?** (A DLQ event that's been sitting there for a week is a DLQ event that nobody's looking at.)
4. **What's the distribution of error types?** (One dominant error type suggests a single root cause. Many error types suggest broader problems.)
5. **Are events being resolved?** (If the inflow exceeds the outflow, the DLQ is growing. That's a problem.)

### Alert Fatigue Warning

Be judicious with alerts. A DLQ that receives one event per hour is normal wear and tear in a large system. A DLQ that receives one hundred events per minute is an incident. Set your thresholds based on your system's normal behavior, not on the theoretical ideal of zero DLQ events.

```python
# Alert logic — alert on anomalies, not on absolute counts
class DLQAlertEvaluator:
    def evaluate(self, current_rate, baseline_rate):
        if current_rate > baseline_rate * 5:
            return Alert.CRITICAL, f"DLQ rate is 5x above baseline ({current_rate}/min vs {baseline_rate}/min)"
        elif current_rate > baseline_rate * 2:
            return Alert.WARNING, f"DLQ rate is elevated ({current_rate}/min vs {baseline_rate}/min)"
        else:
            return Alert.OK, None
```

---

## Summary

Error handling in event-driven systems is not a feature you add at the end. It's a fundamental architectural concern that shapes your design from day one.

The essential lessons:

1. **Accept at-least-once delivery.** Build idempotent consumers. Stop chasing the exactly-once unicorn for anything that touches external systems.
2. **Categorize errors early.** Distinguish between transient failures (retry) and deterministic failures (DLQ immediately). Don't waste time retrying poison pills.
3. **Use exponential backoff with jitter.** Always. No exceptions. The thundering herd is real and it is not your friend.
4. **Design your DLQs like first-class citizens.** Rich error metadata, clear naming conventions, monitoring and alerting, resolution workflows.
5. **Use the transactional outbox pattern** for reliable publishing. The dual-write problem will bite you; it's a matter of when, not if.
6. **Plan for saga compensation failures.** Sometimes the undo fails too. Have a manual fallback.
7. **Monitor your DLQs.** An unmonitored DLQ is a lie you're telling yourself about system reliability.

The difference between a fragile event-driven system and a resilient one isn't the happy path — it's how thoroughly you've thought about everything that can go wrong. And in a distributed system, the things that can go wrong are limited only by your imagination and Murphy's law.
