# Observability and Debugging

Congratulations. You've built an event-driven system. Events flow between services, business logic executes asynchronously, and everything works beautifully — until it doesn't, and you spend four hours staring at six different log aggregators trying to figure out why a customer's order vanished into the void between the payment service and the fulfillment service.

Debugging event-driven systems is a fundamentally different discipline from debugging monolithic or synchronous systems. The tools are different, the mindset is different, and the difficulty level is — let's be diplomatic — elevated. This chapter covers the observability practices that will save you from despair, or at least reduce the despair to manageable levels.

---

## Why Traditional Debugging Fails in Event-Driven Systems

In a monolith, you can set a breakpoint, step through code, and watch a request flow from entry point to database and back. The execution is linear, the state is local, and the call stack tells you everything you need to know.

In an event-driven system, none of this is true:

1. **There is no call stack.** An event is published by Service A, consumed by Service B (maybe minutes later), which publishes another event consumed by Service C. The "stack" spans processes, machines, and time. Your debugger can't step across a Kafka topic.

2. **Execution is non-linear.** A single incoming event might fan out to ten consumers. Each consumer might publish additional events. The execution graph is a DAG, not a stack.

3. **Time is a variable.** In a synchronous system, cause and effect are milliseconds apart. In an async system, they might be seconds, minutes, or hours apart. The event that caused a failure at 3 PM might have been published at 11 AM. Good luck finding that in your logs.

4. **State is distributed.** The full state of a business process is spread across multiple services' databases, multiple topic partitions, and multiple consumer group offsets. No single service has the complete picture.

5. **Reproduction is hard.** You can't just "replay the request" because the system state has changed since the original event was published. Other events have been processed, database rows have been modified, external services have been called. The window of reproduction closed before you even knew there was a bug.

Traditional logging — printing "processing order ord-123" in each service — gives you fragments. What you need is a way to stitch those fragments together into a coherent narrative. That's observability.

---

## The Three Pillars: Logs, Metrics, Traces

The observability community has settled on three complementary signal types. You need all three. Skipping one is like removing a leg from a three-legged stool — technically possible to balance, but you'll fall eventually.

### Logs: What Happened

Structured logs are the foundation. Not `printf` debugging, not unstructured text that you'll regex later — structured, machine-parseable log events with consistent fields.

```json
{
  "timestamp": "2025-03-15T14:22:33.456Z",
  "level": "INFO",
  "service": "fulfillment-service",
  "correlationId": "corr-abc-123",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7",
  "eventType": "OrderCreated",
  "eventId": "evt-789",
  "orderId": "ord-456",
  "message": "Processing order for fulfillment",
  "durationMs": 23
}
```

**Non-negotiable fields in every log line:**
- `correlationId`: ties together all logs for a single business operation.
- `traceId` and `spanId`: ties into distributed tracing (more on this below).
- `service`: which service emitted this log.
- `eventType` and `eventId`: which event triggered this work.
- `timestamp`: with timezone. Always UTC. Fight me.

### Metrics: How It's Going

Metrics are aggregated numerical data over time. They answer "how many," "how fast," and "how broken."

Essential event-driven metrics:

```
# Producer metrics
events_published_total{topic, event_type}           # Counter
event_publish_duration_seconds{topic}                # Histogram
event_publish_errors_total{topic, error_type}        # Counter

# Consumer metrics
events_consumed_total{topic, consumer_group}         # Counter
event_processing_duration_seconds{topic, event_type} # Histogram
event_processing_errors_total{topic, error_type}     # Counter
consumer_lag{topic, partition, consumer_group}        # Gauge

# Broker metrics
topic_message_count{topic}                           # Gauge
partition_offset_latest{topic, partition}             # Gauge

# DLQ metrics
dlq_events_total{source_topic, error_type}           # Counter
dlq_events_pending{source_topic}                     # Gauge
```

The two most important metrics in any event-driven system: **event processing duration** (is anything getting slow?) and **consumer lag** (is anything falling behind?). If you monitor nothing else, monitor these.

### Traces: The Journey

A distributed trace represents the end-to-end journey of a request through multiple services. In an event-driven system, it represents the journey of a business operation through multiple event-processing steps.

A trace is composed of spans, each representing a unit of work:

```
Trace: 4bf92f3577b34da6a3ce929d0e0e4736

[Span 1: api-gateway] POST /orders (12ms)
    └── [Span 2: order-service] CreateOrder (8ms)
         └── [Span 3: order-service] PublishOrderCreated (3ms)
              └── [Span 4: payment-service] ProcessPayment (45ms)
                   ├── [Span 5: payment-service] ChargeCard (40ms)
                   └── [Span 6: payment-service] PublishPaymentProcessed (2ms)
                        └── [Span 7: fulfillment-service] CreateShipment (15ms)
                             └── [Span 8: fulfillment-service] PublishShipmentCreated (2ms)
```

The challenge in event-driven systems is that spans 3 and 4 are separated by a Kafka topic. The payment service doesn't receive an HTTP call from the order service — it receives an event from a topic. The trace context must be propagated *through the event* for the trace to remain connected.

---

## Correlation IDs: Threading Context Through Async Flows

A correlation ID is a unique identifier generated at the beginning of a business operation and carried through every subsequent event and service call. It's the thread that lets you pull on one end and unravel the entire operation.

### Generating Correlation IDs

The correlation ID is typically generated at the system boundary — the API gateway, the initial event producer, or whatever first receives the business request:

```python
import uuid

class APIGateway:
    def handle_request(self, request):
        # Generate or extract correlation ID
        correlation_id = request.headers.get(
            'X-Correlation-ID',
            str(uuid.uuid4())
        )

        # Pass to downstream service
        response = order_service.create_order(
            order_data=request.body,
            correlation_id=correlation_id
        )

        return response
```

### Propagating Through Events

The correlation ID must travel with the event. Put it in the event metadata, not the payload — it's infrastructure context, not business data.

```python
class OrderService:
    def create_order(self, order_data, correlation_id):
        order = Order.create(order_data)

        event = {
            'metadata': {
                'eventId': str(uuid.uuid4()),
                'eventType': 'OrderCreated',
                'correlationId': correlation_id,
                'causationId': None,  # This is the root event
                'timestamp': datetime.utcnow().isoformat(),
                'source': 'order-service'
            },
            'payload': order.to_dict()
        }

        producer.send('orders', key=order.id, value=event)
        return order
```

### The Causation Chain

Beyond correlation IDs, maintain **causation IDs** to track which event caused which. The causation ID of an event is the event ID of the event that triggered its creation.

```python
class PaymentService:
    def handle_order_created(self, event):
        correlation_id = event['metadata']['correlationId']
        causing_event_id = event['metadata']['eventId']

        # Process payment...
        payment = process_payment(event['payload'])

        # Publish with causation chain
        payment_event = {
            'metadata': {
                'eventId': str(uuid.uuid4()),
                'eventType': 'PaymentProcessed',
                'correlationId': correlation_id,       # Same correlation ID
                'causationId': causing_event_id,       # Points to OrderCreated
                'timestamp': datetime.utcnow().isoformat(),
                'source': 'payment-service'
            },
            'payload': payment.to_dict()
        }

        producer.send('payments', key=payment.order_id, value=payment_event)
```

With this chain, you can reconstruct the complete event lineage for any business operation:

```
OrderCreated (evt-001, causation: null, correlation: corr-abc)
  └── PaymentProcessed (evt-002, causation: evt-001, correlation: corr-abc)
       └── InventoryReserved (evt-003, causation: evt-002, correlation: corr-abc)
            └── ShipmentCreated (evt-004, causation: evt-003, correlation: corr-abc)
```

### Kafka Header Propagation

In Kafka, use message headers for metadata propagation rather than embedding it in the payload:

```python
from confluent_kafka import Producer

def publish_event(producer, topic, key, payload, correlation_id, causation_id, trace_context):
    headers = [
        ('correlation-id', correlation_id.encode('utf-8')),
        ('causation-id', causation_id.encode('utf-8') if causation_id else b''),
        ('event-type', payload['eventType'].encode('utf-8')),
        ('trace-parent', trace_context.encode('utf-8')),
        ('source-service', 'order-service'.encode('utf-8')),
    ]

    producer.produce(
        topic=topic,
        key=key.encode('utf-8'),
        value=json.dumps(payload).encode('utf-8'),
        headers=headers
    )
    producer.flush()
```

Headers have several advantages over payload-embedded metadata: they're accessible without deserializing the event, they can be read by infrastructure tooling (monitoring, routing) that doesn't understand the payload schema, and they keep infrastructure concerns separate from business data.

---

## Distributed Tracing with OpenTelemetry

OpenTelemetry (OTel) is the industry-standard framework for distributed tracing (and metrics, and logs, but tracing is where it shines in event-driven systems). The key challenge is propagating trace context through message brokers, which weren't designed with tracing in mind.

### The W3C Trace Context Standard

The `traceparent` header carries trace context in a standardized format:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
              |  |                                  |                |
              |  trace-id (128 bit)                 span-id (64 bit)|
              version                                              flags (sampled)
```

### Producer-Side: Injecting Trace Context

```python
from opentelemetry import trace
from opentelemetry.context import attach, detach
from opentelemetry.trace.propagation import get_current_span
from opentelemetry.propagators import inject

tracer = trace.get_tracer("order-service")

def publish_order_created(order):
    with tracer.start_as_current_span("publish_order_created") as span:
        span.set_attribute("messaging.system", "kafka")
        span.set_attribute("messaging.destination", "orders")
        span.set_attribute("messaging.destination_kind", "topic")
        span.set_attribute("order.id", order.id)

        # Inject trace context into carrier (event headers)
        headers = {}
        inject(headers)  # Injects traceparent and tracestate

        # Convert to Kafka header format
        kafka_headers = [
            (k, v.encode('utf-8')) for k, v in headers.items()
        ]

        producer.produce(
            topic='orders',
            key=order.id,
            value=serialize(order.to_event()),
            headers=kafka_headers
        )
```

### Consumer-Side: Extracting and Continuing the Trace

```python
from opentelemetry.propagators import extract

tracer = trace.get_tracer("payment-service")

def handle_message(message):
    # Extract trace context from Kafka headers
    carrier = {
        h[0]: h[1].decode('utf-8')
        for h in message.headers() or []
    }
    ctx = extract(carrier)

    # Create a new span linked to the producer's span
    with tracer.start_as_current_span(
        "process_order_created",
        context=ctx,
        kind=trace.SpanKind.CONSUMER,
        attributes={
            "messaging.system": "kafka",
            "messaging.source": message.topic(),
            "messaging.message_id": message.key().decode('utf-8'),
            "messaging.kafka.partition": message.partition(),
            "messaging.kafka.offset": message.offset(),
        }
    ) as span:
        try:
            process_payment(message.value())
            span.set_status(trace.StatusCode.OK)
        except Exception as e:
            span.set_status(trace.StatusCode.ERROR, str(e))
            span.record_exception(e)
            raise
```

### The Produce-Consume Link

The connection between the producer span and the consumer span is what makes distributed tracing valuable in event-driven systems. Without it, you have two disconnected traces. With it, you have a continuous narrative from "user clicked 'Place Order'" to "warehouse picked the item."

OpenTelemetry supports two linking strategies:

1. **Parent-child:** The consumer span is a child of the producer span. This creates a single trace that includes both the producing and consuming work. Simple and intuitive, but can create very wide traces if one event triggers many consumers.

2. **Links:** The consumer span is a new trace root with a *link* to the producer span. This keeps traces manageable in fan-out scenarios but requires tooling that can follow links across traces.

```python
# Link-based approach for fan-out scenarios
from opentelemetry.trace import Link

def handle_message_with_link(message):
    carrier = extract_headers(message)
    producer_context = extract(carrier)
    producer_span_context = trace.get_current_span(producer_context).get_span_context()

    with tracer.start_as_current_span(
        "process_event",
        links=[Link(producer_span_context)],
        kind=trace.SpanKind.CONSUMER
    ) as span:
        process(message)
```

---

## Event Lineage and Causation Chains

Distributed tracing gives you the *how* — which services participated and how long they took. Event lineage gives you the *what* — which events caused which other events, forming a directed graph of causation.

### Building an Event Lineage Store

```python
class EventLineageStore:
    """Stores and queries event causation relationships."""

    def __init__(self, db):
        self.db = db

    def record_event(self, event_id, event_type, correlation_id,
                     causation_id, source_service, timestamp):
        self.db.insert('event_lineage', {
            'event_id': event_id,
            'event_type': event_type,
            'correlation_id': correlation_id,
            'causation_id': causation_id,
            'source_service': source_service,
            'timestamp': timestamp
        })

    def get_full_chain(self, correlation_id):
        """Get all events in a business operation, ordered by time."""
        return self.db.query(
            "SELECT * FROM event_lineage "
            "WHERE correlation_id = %s "
            "ORDER BY timestamp",
            [correlation_id]
        )

    def get_descendants(self, event_id):
        """Get all events caused (directly or transitively) by an event."""
        return self.db.query("""
            WITH RECURSIVE chain AS (
                SELECT * FROM event_lineage WHERE event_id = %s
                UNION ALL
                SELECT el.* FROM event_lineage el
                JOIN chain c ON el.causation_id = c.event_id
            )
            SELECT * FROM chain ORDER BY timestamp
        """, [event_id])

    def get_root_cause(self, event_id):
        """Walk up the causation chain to find the originating event."""
        return self.db.query("""
            WITH RECURSIVE chain AS (
                SELECT * FROM event_lineage WHERE event_id = %s
                UNION ALL
                SELECT el.* FROM event_lineage el
                JOIN chain c ON el.event_id = c.causation_id
            )
            SELECT * FROM chain WHERE causation_id IS NULL
        """, [event_id])
```

### Visualizing Event Flow

A lineage store lets you answer questions that are otherwise nearly impossible:

- "Show me every event that resulted from this customer's order" (get descendants of the root event).
- "This payment failed — what started the process?" (walk up the causation chain).
- "These two services are both updating this entity — is there a shared upstream event?" (find common ancestor).

```
Query: get_full_chain('corr-abc-123')

Timeline:
14:22:33.100  [order-service]       OrderCreated (evt-001)
14:22:33.250  [payment-service]     PaymentProcessed (evt-002, caused by evt-001)
14:22:33.400  [inventory-service]   InventoryReserved (evt-003, caused by evt-002)
14:22:33.500  [notification-service] CustomerNotified (evt-004, caused by evt-002)
14:22:34.100  [fulfillment-service]  ShipmentCreated (evt-005, caused by evt-003)
```

---

## Consumer Lag Monitoring: The Canary in the Coal Mine

Consumer lag is the difference between the latest offset in a topic partition and the offset that a consumer group has committed. It tells you how far behind a consumer is — how many unprocessed events are waiting.

### Why Lag Matters

A small, stable lag is normal. Events arrive faster than you process them during bursts, and you catch up during lulls. This is fine.

A growing lag is a problem. It means your consumer is falling further behind, which means:
- Events are being processed with increasing delay (SLA violations).
- If the lag grows past the topic's retention period, events will be deleted before they're processed (data loss).
- The consumer is likely degraded or stuck (processing too slowly, or blocked by a poison pill).

### Monitoring Lag

```python
# Using confluent_kafka's AdminClient to check consumer lag
from confluent_kafka.admin import AdminClient

def check_consumer_lag(bootstrap_servers, consumer_group, topic):
    admin = AdminClient({'bootstrap.servers': bootstrap_servers})

    # Get the latest offsets for each partition
    topic_metadata = admin.list_topics(topic)
    partitions = topic_metadata.topics[topic].partitions

    total_lag = 0
    for partition_id in partitions:
        tp = TopicPartition(topic, partition_id)

        # Get the committed offset for this consumer group
        committed = admin.list_consumer_group_offsets(consumer_group, [tp])
        committed_offset = committed[tp].offset

        # Get the latest offset (high watermark)
        latest_offset = consumer.get_watermark_offsets(tp)[1]

        lag = latest_offset - committed_offset
        total_lag += lag

        if lag > 10000:
            alert(f"High lag on {topic}/{partition_id}: "
                  f"{lag} events behind (consumer: {consumer_group})")

    return total_lag
```

### Lag-Based Alerts

Set up tiered alerts:

```yaml
# Alert rules for consumer lag
alerts:
  - name: consumer_lag_warning
    condition: "consumer_lag > 1000 for 5 minutes"
    severity: warning
    message: "Consumer group {consumer_group} is {lag} events behind on {topic}"

  - name: consumer_lag_critical
    condition: "consumer_lag > 10000 for 10 minutes"
    severity: critical
    message: "Consumer group {consumer_group} is severely behind ({lag} events). Check for stuck consumers or poison pills."

  - name: consumer_lag_approaching_retention
    condition: "consumer_lag_time_seconds > (retention_seconds * 0.8)"
    severity: critical
    message: "Consumer group {consumer_group} lag is approaching topic retention. DATA LOSS IMMINENT."
```

The last alert is the one that should wake people up at night. When lag (measured in time, not just events) approaches the topic's retention period, events will start being deleted before they're consumed. This is irrecoverable data loss.

### Lag as a Health Signal

Consumer lag is the single most useful health signal for an event-driven system. Trend it, dashboard it, alert on it. A system where lag is stable and low is healthy. A system where lag is growing is sick, even if nothing else looks wrong yet.

---

## Dead Letter Queue Dashboards

We covered DLQs in Chapter 5. Here's the observability angle.

### What Your DLQ Dashboard Needs

```
┌─────────────────────────────────────────────────────┐
│ Dead Letter Queue Overview                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│ Total Pending Events: 47          Oldest: 2h 15m    │
│                                                     │
│ Inflow (last hour): 12/hr        Outflow: 8/hr     │
│                                                     │
│ By Source Topic:                                    │
│   orders.order-created         │████████░░│ 23      │
│   payments.payment-processed   │███░░░░░░░│ 11      │
│   inventory.stock-updated      │██░░░░░░░░│ 8       │
│   shipping.label-created       │█░░░░░░░░░│ 5       │
│                                                     │
│ By Error Type:                                      │
│   TimeoutError                 │██████░░░░│ 19      │
│   ValidationError              │████░░░░░░│ 14      │
│   SerializationError           │███░░░░░░░│ 9       │
│   Unknown                      │█░░░░░░░░░│ 5       │
│                                                     │
│ Trend (24h):                                        │
│   ▃▃▂▂▁▁▁▂▃▅▇█▇▅▃▃▂▂▁▁▁▁▁                         │
│   ^                 ^                               │
│   midnight          noon                            │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### DLQ Event Explorer

Beyond aggregate metrics, you need the ability to drill into individual events:

```python
class DLQExplorer:
    """REST API for investigating DLQ events."""

    def get_events(self, source_topic=None, error_type=None,
                   since=None, limit=50):
        query = "SELECT * FROM dlq_events WHERE 1=1"
        params = []

        if source_topic:
            query += " AND source_topic = %s"
            params.append(source_topic)
        if error_type:
            query += " AND error_type = %s"
            params.append(error_type)
        if since:
            query += " AND failed_at > %s"
            params.append(since)

        query += " ORDER BY failed_at DESC LIMIT %s"
        params.append(limit)

        return self.db.query(query, params)

    def get_event_detail(self, dlq_event_id):
        event = self.db.get('dlq_events', dlq_event_id)

        # Enrich with lineage data
        event['lineage'] = self.lineage_store.get_full_chain(
            event['correlation_id']
        )

        # Include related successful events for context
        event['related_events'] = self.get_related_events(
            event['correlation_id']
        )

        return event

    def replay_event(self, dlq_event_id):
        event = self.db.get('dlq_events', dlq_event_id)
        original_topic = event['source_topic']
        original_event = event['original_event']

        self.producer.send(original_topic, original_event)
        self.db.update('dlq_events', dlq_event_id,
                       {'status': 'replayed', 'replayed_at': datetime.utcnow()})
```

---

## Event Replay and Time-Travel Debugging

One of the genuine advantages of event-driven architecture (specifically, event sourcing or log-based systems like Kafka) is the ability to replay events. This is your time machine.

### Replaying for Debugging

When a consumer produces incorrect results, you can:

1. Reset the consumer group offset to a point before the bug.
2. Fix the bug and deploy the new consumer.
3. Replay all events from the reset point.

```bash
# Reset consumer group to a specific timestamp
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group fulfillment-service \
  --topic orders \
  --reset-offsets --to-datetime 2025-03-15T10:00:00.000 \
  --execute
```

**Warnings:**
- This replays *all* events from that timestamp, not just the one you're debugging. If your consumer has side effects (sends emails, charges credit cards), you need to disable those during replay or use a separate consumer group.
- Replay with side effects is how you email a customer 47 times. Ask me how I know.

### The Replay Consumer Pattern

A safer approach is a dedicated replay consumer that processes events without side effects, just to reconstruct state:

```python
class ReplayConsumer:
    """Replays events to reconstruct state at a point in time."""

    def __init__(self, topic, target_timestamp):
        self.consumer = KafkaConsumer(
            topic,
            group_id=f'replay-{uuid4()}',  # Unique group — won't affect production
            auto_offset_reset='earliest',
            enable_auto_commit=False
        )
        self.target_timestamp = target_timestamp
        self.state = {}

    def replay(self, entity_id=None):
        for message in self.consumer:
            if message.timestamp > self.target_timestamp:
                break

            event = deserialize(message.value)

            if entity_id and event.get('entityId') != entity_id:
                continue

            self._apply(event)

        return self.state

    def _apply(self, event):
        entity_id = event['entityId']
        if entity_id not in self.state:
            self.state[entity_id] = {}

        # Apply event to state (event sourcing projection)
        event_type = event['metadata']['eventType']
        handler = getattr(self, f'_handle_{event_type}', None)
        if handler:
            handler(event, self.state[entity_id])

    def _handle_OrderCreated(self, event, state):
        state.update(event['payload'])
        state['status'] = 'created'

    def _handle_OrderShipped(self, event, state):
        state['status'] = 'shipped'
        state['shippingInfo'] = event['payload']['shippingInfo']
```

### Time-Travel Queries

If you're using event sourcing, you can reconstruct the state of any entity at any point in time:

```python
def get_order_state_at(order_id, target_time):
    """What did this order look like at the given timestamp?"""
    events = event_store.get_events(
        aggregate_id=order_id,
        up_to=target_time
    )

    state = {}
    for event in events:
        apply_event(state, event)

    return state

# "What was the order state when the payment was processed?"
payment_event = event_store.get_event('evt-002')
order_state = get_order_state_at('ord-456', payment_event['timestamp'])
```

This is extremely powerful for debugging: "The fulfillment service says the order had no shipping address, but the customer definitely entered one. Let's see what the order looked like at the exact moment the fulfillment event was published."

---

## Debugging Patterns: The "Event Detective" Workflow

When something goes wrong in an event-driven system, here's a systematic approach:

### Step 1: Start with the Symptom

"Customer says their order confirmation email never arrived."

### Step 2: Find the Correlation ID

Look up the order in your system, find the correlation ID for the business operation:

```sql
SELECT correlation_id FROM orders WHERE order_id = 'ord-789';
-- Result: corr-def-456
```

### Step 3: Pull the Event Chain

Query your event lineage store for all events in this correlation:

```sql
SELECT event_id, event_type, source_service, timestamp, causation_id
FROM event_lineage
WHERE correlation_id = 'corr-def-456'
ORDER BY timestamp;
```

```
evt-101  OrderCreated          order-service        14:22:33.100  NULL
evt-102  PaymentProcessed      payment-service      14:22:33.250  evt-101
evt-103  InventoryReserved     inventory-service    14:22:33.400  evt-102
evt-104  ShipmentCreated       fulfillment-service  14:22:34.100  evt-103
```

Notice anything? There's no `CustomerNotified` event. The notification service never fired.

### Step 4: Check the Consumer

Is the notification service consuming from the right topic? Is it healthy?

```bash
# Check consumer lag for notification-service
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group notification-service --describe
```

Output shows the notification service is caught up on the `payments` topic. So it received `PaymentProcessed` but didn't emit `CustomerNotified`. Why?

### Step 5: Check the Logs

```
# Search logs for this correlation ID in the notification service
correlationId:corr-def-456 AND service:notification-service
```

```json
{
  "timestamp": "2025-03-15T14:22:33.260Z",
  "level": "ERROR",
  "service": "notification-service",
  "correlationId": "corr-def-456",
  "eventId": "evt-102",
  "message": "Failed to render email template: missing field 'customerEmail'",
  "error": "KeyError: 'customerEmail'"
}
```

Found it. The `PaymentProcessed` event didn't include the customer's email, and the notification service failed. But where did the failure go?

### Step 6: Check the DLQ

```sql
SELECT * FROM dlq_events
WHERE correlation_id = 'corr-def-456'
AND source_service = 'notification-service';
```

If the event is in the DLQ, you know the failure was caught. If it's not, the failure was swallowed silently — a logging-only error handler with no DLQ routing. That's its own bug.

### Step 7: Fix and Replay

Fix the notification service to handle the missing field (or fix the payment service to include it), then replay the event from the DLQ.

This entire workflow — symptom, correlation, chain, consumer, logs, DLQ, fix — should take minutes, not hours. It takes minutes only if you've invested in the tooling described in this chapter.

---

## Common Observability Anti-Patterns

### 1. The Log Volcano

Logging everything at DEBUG level in production because "we might need it." You're generating terabytes of logs, your log aggregator is expensive and slow, and the signal-to-noise ratio is atrocious. Nobody can find anything.

**Fix:** Log at INFO for normal operations, WARN for recoverable anomalies, ERROR for failures. Use structured logging with consistent fields. Enable DEBUG temporarily and selectively during active investigations.

### 2. The Metric Desert

No metrics, or only infrastructure metrics (CPU, memory, disk). You know the machine is healthy but have no idea if the application is working correctly.

**Fix:** Instrument your business logic. Every event processed, every external call, every state transition should have metrics. Focus on the RED method: Rate, Errors, Duration.

### 3. Orphan Traces

Traces that stop at the broker boundary because nobody propagated the trace context through events. You have perfect visibility within each service and zero visibility across services — which is exactly where you need it most.

**Fix:** Make trace context propagation a standard part of your event production and consumption libraries. Don't rely on individual developers to remember.

### 4. The Missing Correlation ID

Some events have correlation IDs, some don't. Some services propagate them, some generate new ones. The result is fragmented event chains that you can't stitch together.

**Fix:** Correlation ID propagation must be enforced at the framework level, not the application level. Use middleware or interceptors that automatically extract and inject correlation IDs. Reject events that don't have one.

### 5. Alert Noise

Alerting on every DLQ event, every momentary lag spike, every transient error. Your on-call engineers learn to ignore alerts, which means they ignore the real ones too.

**Fix:** Alert on sustained anomalies, not individual events. Use anomaly detection rather than static thresholds where possible. Have distinct channels for warnings (investigate when convenient) and critical alerts (investigate now).

### 6. The Post-Mortem Information Gap

Something fails on Friday, you investigate on Monday, and the relevant logs have been rotated, the metrics resolution is too coarse to see the spike, and the trace was sampled away.

**Fix:** Retain high-resolution data for at least 7 days. Keep aggregated data for months. Never sample traces for error conditions — always capture traces for failed operations at 100%.

### 7. Observing Only the Happy Path

All your dashboards show throughput and latency for successful events. You have no visibility into failures, retries, or DLQ activity. The system looks green while 2% of events are silently failing.

**Fix:** Failure metrics should be as prominent as success metrics. A dashboard that only shows success rate is a lie of omission.

---

## Tools of the Trade

A brief, opinionated survey of the observability ecosystem for event-driven systems.

### Distributed Tracing

**Jaeger**: Open-source, CNCF graduated project. Good for Kubernetes-native deployments. Supports OpenTelemetry natively. The UI is functional if not beautiful. Scales well with Elasticsearch or Cassandra as the storage backend.

**Zipkin**: The original open-source distributed tracing tool. Simpler than Jaeger, with a cleaner UI. Good for smaller deployments. Less active development than Jaeger.

**Grafana Tempo**: A cost-effective trace storage backend that integrates with Grafana. Uses object storage (S3, GCS) instead of dedicated databases, which dramatically reduces cost at scale. The tradeoff is query latency — searching for traces is slower than Jaeger, but looking up a trace by ID is fast.

**Datadog APM**: SaaS, fully managed, excellent UI, deep integration with metrics and logs. Expensive. The trace-to-log correlation is genuinely good. If you're already paying for Datadog, the APM is worth enabling.

### Metrics

**Prometheus + Grafana**: The open-source standard. Prometheus scrapes metrics, Grafana visualizes them. Grafana's dashboarding is best-in-class. Prometheus's pull-based model works well for Kubernetes but can be awkward for short-lived processes.

**Datadog**: SaaS metrics with excellent tagging and correlation. The DogStatsD agent is easy to integrate. The cost scales with the number of custom metrics, which can get expensive.

### Logging

**ELK Stack (Elasticsearch, Logstash, Kibana)**: The classic. Powerful but operationally heavy. Elasticsearch clusters require care and feeding.

**Grafana Loki**: Log aggregation that indexes labels, not content. Much cheaper than Elasticsearch for high-volume logs. Query language (LogQL) is good. Full-text search is slower than Elasticsearch.

**Datadog Logs**: SaaS, integrates with traces and metrics. The log-to-trace correlation feature is excellent for event-driven debugging. The pricing is per-gigabyte ingested, which concentrates the mind wonderfully on what you actually need to log.

### Kafka-Specific Observability

**Kafka Manager / CMAK (Cluster Manager for Apache Kafka)**: Basic cluster management and consumer lag monitoring. Aging but functional.

**Kafka UI / Conduktor / AKHQ**: Modern UIs for exploring topics, consumer groups, and messages. Essential for development and debugging.

**Burrow**: LinkedIn's consumer lag monitoring tool. Evaluates lag as a sliding window rather than a point-in-time snapshot, which produces more meaningful alerts. Highly recommended if you're running Kafka.

---

## Putting It All Together

Here's a complete example of an instrumented event consumer that incorporates correlation IDs, trace propagation, structured logging, metrics, and error handling:

```python
import json
import time
import uuid
import logging
from opentelemetry import trace, metrics
from opentelemetry.propagators import extract
from prometheus_client import Counter, Histogram

# Metrics
events_processed = Counter(
    'events_processed_total',
    'Total events processed',
    ['topic', 'event_type', 'status']
)
processing_duration = Histogram(
    'event_processing_duration_seconds',
    'Event processing duration',
    ['topic', 'event_type']
)

tracer = trace.get_tracer("fulfillment-service")
logger = logging.getLogger("fulfillment-service")

class InstrumentedConsumer:
    def __init__(self, consumer, processor, dlq_producer):
        self.consumer = consumer
        self.processor = processor
        self.dlq_producer = dlq_producer

    def run(self):
        for message in self.consumer:
            self._handle_message(message)

    def _handle_message(self, message):
        # Extract headers
        headers = {h[0]: h[1].decode('utf-8') for h in (message.headers() or [])}
        correlation_id = headers.get('correlation-id', str(uuid.uuid4()))
        event_type = headers.get('event-type', 'unknown')

        # Extract trace context
        ctx = extract(headers)

        # Start a span linked to the producer
        with tracer.start_as_current_span(
            f"process_{event_type}",
            context=ctx,
            kind=trace.SpanKind.CONSUMER,
            attributes={
                "messaging.system": "kafka",
                "messaging.source": message.topic(),
                "messaging.kafka.partition": message.partition(),
                "messaging.kafka.offset": message.offset(),
                "correlation.id": correlation_id,
                "event.type": event_type,
            }
        ) as span:
            start_time = time.time()

            # Structured log: event received
            logger.info("Event received", extra={
                'correlationId': correlation_id,
                'eventType': event_type,
                'topic': message.topic(),
                'partition': message.partition(),
                'offset': message.offset(),
            })

            try:
                event = json.loads(message.value())
                self.processor.handle(event, correlation_id)

                # Success metrics and logging
                duration = time.time() - start_time
                events_processed.labels(
                    topic=message.topic(),
                    event_type=event_type,
                    status='success'
                ).inc()
                processing_duration.labels(
                    topic=message.topic(),
                    event_type=event_type
                ).observe(duration)

                span.set_status(trace.StatusCode.OK)
                logger.info("Event processed successfully", extra={
                    'correlationId': correlation_id,
                    'eventType': event_type,
                    'durationMs': int(duration * 1000),
                })

            except Exception as e:
                # Error metrics and logging
                duration = time.time() - start_time
                events_processed.labels(
                    topic=message.topic(),
                    event_type=event_type,
                    status='error'
                ).inc()

                span.set_status(trace.StatusCode.ERROR, str(e))
                span.record_exception(e)

                logger.error("Event processing failed", extra={
                    'correlationId': correlation_id,
                    'eventType': event_type,
                    'error': str(e),
                    'errorType': type(e).__name__,
                    'durationMs': int(duration * 1000),
                })

                # Send to DLQ with full context
                self._send_to_dlq(message, e, correlation_id)

    def _send_to_dlq(self, message, error, correlation_id):
        import traceback

        dlq_event = {
            'originalEvent': message.value().decode('utf-8'),
            'error': {
                'message': str(error),
                'type': type(error).__name__,
                'stackTrace': traceback.format_exc(),
            },
            'context': {
                'correlationId': correlation_id,
                'topic': message.topic(),
                'partition': message.partition(),
                'offset': message.offset(),
                'failedAt': time.time(),
            }
        }

        self.dlq_producer.send(
            f"{message.topic()}.dlq",
            json.dumps(dlq_event).encode('utf-8')
        )
```

This is not a small amount of code. That's the honest truth about observability in event-driven systems: it's substantial, it's pervasive, and it needs to be baked into your event processing framework, not sprinkled on top by individual developers.

---

## Summary

Observability in event-driven systems is not a nice-to-have; it's a prerequisite for operating with confidence. The async, distributed nature of these systems means that without deliberate investment in observability, you are flying blind.

The essentials:

1. **Correlation IDs are mandatory.** Every event, every log line, every trace. No exceptions. Enforce this at the framework level.
2. **Propagate trace context through events.** Use OpenTelemetry and the W3C Trace Context standard. Instrument your producer and consumer libraries once, and every service benefits.
3. **Build event lineage tracking.** Causation chains let you reconstruct the complete story of a business operation. This is your primary debugging tool.
4. **Monitor consumer lag religiously.** It's the single best health indicator for an event-driven system. Alert on sustained growth, not momentary spikes.
5. **Instrument your DLQs.** Dashboard them, alert on them, build tooling to investigate and replay failed events.
6. **Invest in replay capability.** The ability to time-travel through your event history is one of the few genuine advantages event-driven systems have over request-response architectures. Use it.
7. **Standardize your observability stack** across all services. Fragmented tooling means fragmented visibility, which means fragmented debugging.

The investment is significant. The alternative — debugging a production incident by grepping through six services' logs, correlating timestamps by hand, and guessing at causation — is significantly worse. Build the tooling, maintain the discipline, and your future on-call self will be grateful.
