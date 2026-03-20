# Testing Event-Driven Systems

Testing synchronous request-response systems is straightforward. Call a function, get a result, assert on the result. Testing event-driven systems is... not that. You publish an event and then wait. Something might happen. Eventually. Probably. Somewhere else. In a different process. On a different machine. And your test needs to verify that the right thing happened without being able to observe it directly.

If your test suite for an event-driven system looks the same as your test suite for a REST API, one of two things is true: either your system isn't actually event-driven, or your tests aren't actually testing anything.

---

## Why Testing Async Systems Is Fundamentally Different

The core difficulty is that event-driven systems break the temporal coupling between cause and effect. In a synchronous system:

```
request -> processing -> response (all in one call, one thread, one moment)
```

In an event-driven system:

```
publish event -> ??? -> eventually a consumer processes it -> ??? -> maybe a side effect occurs
```

This introduces several testing challenges that don't exist in the synchronous world:

1. **Non-deterministic timing.** When you publish an event, you don't know when it will be consumed. It depends on broker latency, consumer lag, partition assignment, rebalancing, and the phase of the moon.

2. **No return value.** A producer gets an acknowledgment that the event was written to the broker. It does not get confirmation that any consumer processed it, let alone processed it correctly.

3. **Distributed state.** The outcome of processing an event might be a state change in a different service's database, the publication of another event, or a call to an external API. Your test needs to observe state in a different system.

4. **Ordering is conditional.** Events may arrive in order within a partition and out of order across partitions. Your tests need to account for both cases.

5. **Exactly-once is a spectrum.** Your tests need to verify behavior under at-least-once delivery, which means verifying idempotency, which means running the same event through the same consumer multiple times and asserting on the outcome.

6. **Infrastructure dependency.** You can't meaningfully test event-driven behavior without a broker (or a convincing fake). This pushes more of your testing into integration territory.

The test pyramid, that beloved conference slide, starts to look more like a test diamond — a thin layer of unit tests at the bottom, a fat layer of integration tests in the middle, and a thin layer of end-to-end tests at the top.

---

## Unit Testing Event Producers and Consumers in Isolation

Despite everything I just said about the difficulty of testing async systems, unit tests still have a role. The trick is knowing what to test at the unit level and what to push to integration tests.

### Testing Producers

A producer's job is to create a well-formed event and hand it to the broker. The unit test should verify the event creation, not the broker interaction.

```python
# The producer logic, separated from broker interaction
class OrderEventProducer:
    def __init__(self, event_publisher):
        self.event_publisher = event_publisher

    def create_order_placed_event(self, order) -> dict:
        """Create an OrderPlaced event from an Order domain object."""
        return {
            "eventType": "OrderPlaced",
            "eventId": str(uuid.uuid4()),
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "version": 1,
            "data": {
                "orderId": order.id,
                "customerId": order.customer_id,
                "items": [
                    {"sku": item.sku, "quantity": item.quantity, "price": str(item.price)}
                    for item in order.items
                ],
                "totalAmount": str(order.total_amount),
                "currency": order.currency,
            }
        }

    def publish_order_placed(self, order):
        event = self.create_order_placed_event(order)
        self.event_publisher.publish("orders", key=order.id, value=event)
        return event


# Unit test — no broker needed
class TestOrderEventProducer:
    def test_creates_valid_order_placed_event(self):
        order = Order(
            id="ord-123",
            customer_id="cust-456",
            items=[OrderItem(sku="WIDGET-001", quantity=2, price=Decimal("29.99"))],
            total_amount=Decimal("59.98"),
            currency="USD"
        )

        producer = OrderEventProducer(event_publisher=Mock())
        event = producer.create_order_placed_event(order)

        assert event["eventType"] == "OrderPlaced"
        assert event["version"] == 1
        assert event["data"]["orderId"] == "ord-123"
        assert event["data"]["totalAmount"] == "59.98"
        assert len(event["data"]["items"]) == 1
        assert event["data"]["items"][0]["sku"] == "WIDGET-001"

    def test_publish_calls_event_publisher_with_correct_topic_and_key(self):
        mock_publisher = Mock()
        producer = OrderEventProducer(event_publisher=mock_publisher)

        order = Order(id="ord-123", customer_id="cust-456", items=[],
                      total_amount=Decimal("0"), currency="USD")
        producer.publish_order_placed(order)

        mock_publisher.publish.assert_called_once()
        call_args = mock_publisher.publish.call_args
        assert call_args[0][0] == "orders"           # topic
        assert call_args[1]["key"] == "ord-123"       # partition key
```

The key insight: separate event construction from event transmission. The construction logic is pure business logic — testable with unit tests. The transmission is infrastructure interaction — testable with integration tests.

### Testing Consumers

A consumer's job is to receive an event, validate it, and perform some action. The unit test should verify the action logic, assuming a valid event arrives.

```python
# Consumer logic, separated from broker interaction
class OrderEventConsumer:
    def __init__(self, inventory_service, notification_service):
        self.inventory_service = inventory_service
        self.notification_service = notification_service

    def handle_order_placed(self, event: dict):
        """Process an OrderPlaced event."""
        order_data = event["data"]

        # Reserve inventory for each item
        for item in order_data["items"]:
            self.inventory_service.reserve(
                sku=item["sku"],
                quantity=item["quantity"],
                order_id=order_data["orderId"]
            )

        # Notify the customer
        self.notification_service.send_order_confirmation(
            customer_id=order_data["customerId"],
            order_id=order_data["orderId"]
        )


# Unit test — no broker needed
class TestOrderEventConsumer:
    def test_reserves_inventory_for_each_item(self):
        inventory = Mock()
        notifications = Mock()
        consumer = OrderEventConsumer(inventory, notifications)

        event = {
            "eventType": "OrderPlaced",
            "data": {
                "orderId": "ord-123",
                "customerId": "cust-456",
                "items": [
                    {"sku": "WIDGET-001", "quantity": 2, "price": "29.99"},
                    {"sku": "GADGET-002", "quantity": 1, "price": "49.99"},
                ],
            }
        }

        consumer.handle_order_placed(event)

        assert inventory.reserve.call_count == 2
        inventory.reserve.assert_any_call(
            sku="WIDGET-001", quantity=2, order_id="ord-123"
        )
        inventory.reserve.assert_any_call(
            sku="GADGET-002", quantity=1, order_id="ord-123"
        )

    def test_sends_order_confirmation(self):
        inventory = Mock()
        notifications = Mock()
        consumer = OrderEventConsumer(inventory, notifications)

        event = {
            "eventType": "OrderPlaced",
            "data": {
                "orderId": "ord-123",
                "customerId": "cust-456",
                "items": [],
            }
        }

        consumer.handle_order_placed(event)

        notifications.send_order_confirmation.assert_called_once_with(
            customer_id="cust-456", order_id="ord-123"
        )
```

This pattern — extracting the handler logic from the message consumption loop — is the single most important testing technique for event-driven consumers. If your event handler is tangled up with your `KafkaConsumer.poll()` loop, your tests will be tangled up with Kafka too.

---

## Contract Testing with Pact and Similar Tools

Unit tests verify that your producer creates the right shape of event and your consumer handles that shape correctly. But who verifies that the shape the producer creates is the shape the consumer expects?

This is the contract problem, and it's amplified in event-driven systems where producers and consumers are developed by different teams, deployed independently, and communicate only through events that flow through a broker.

### What Is Contract Testing?

A contract test verifies that two systems agree on the format of the messages they exchange, without requiring both systems to be running simultaneously. It's the event-driven equivalent of "did you read the API docs?" except automated, mandatory, and not dependent on anyone actually writing or reading docs.

### Pact for Event-Driven Systems

Pact is the most widely-used contract testing framework. It was originally designed for HTTP APIs but supports message-based interactions through its message pact feature.

```python
# Consumer-side Pact test (consumer defines what it expects)
# This is the "consumer-driven" part — the consumer defines the contract

from pact import MessageConsumer, MessagePact

def test_order_placed_contract():
    pact = MessageConsumer('ShippingService').has_pact_with(
        MessagePact('OrderService'),
        pact_dir='./pacts'
    )

    expected_event = {
        "eventType": "OrderPlaced",
        "version": 1,
        "data": {
            "orderId": Like("ord-12345"),           # any string matching pattern
            "customerId": Like("cust-789"),
            "items": EachLike({
                "sku": Like("WIDGET-001"),
                "quantity": Like(1),                 # any integer
                "price": Like("29.99"),              # any string (decimal)
            }),
            "shippingAddress": {
                "street": Like("123 Main St"),
                "city": Like("Springfield"),
                "state": Like("IL"),
                "zip": Like("62701"),
                "country": Like("US"),
            }
        }
    }

    (pact
        .given("an order exists")
        .expects_to_receive("an OrderPlaced event")
        .with_content(expected_event)
        .with_metadata({"topic": "orders", "contentType": "application/json"}))

    # The handler that processes this event
    with pact:
        handler = OrderEventHandler()
        handler.handle(expected_event)

    # Pact writes a contract file (pact JSON) to ./pacts/
    # This file is shared with the OrderService (provider) for verification
```

```python
# Producer-side Pact verification (provider verifies it meets the contract)

from pact import MessageProvider

def test_order_service_satisfies_shipping_contract():
    provider = MessageProvider(
        provider='OrderService',
        consumer='ShippingService',
        pact_dir='./pacts'     # read the contract the consumer defined
    )

    def order_placed_message_factory():
        """
        Produce an actual OrderPlaced event using real producer code.
        Pact will compare this against the consumer's expectations.
        """
        order = create_test_order()
        producer = OrderEventProducer(event_publisher=Mock())
        return producer.create_order_placed_event(order)

    provider.add_message_interaction(
        description="an OrderPlaced event",
        provider_states=[{"name": "an order exists"}],
        message_factory=order_placed_message_factory,
    )

    # Verify: does the actual event match what the consumer expects?
    provider.verify()
```

### The Pact Broker

In practice, consumer pact files need to get to the provider somehow. The Pact Broker is a central service that stores and shares pacts:

```
Consumer (Shipping) --[publishes pact]--> Pact Broker <--[fetches pact]-- Provider (Orders)
                                               |
                                         [verification results]
                                               |
                                    CI pipeline: "can I deploy?"
```

The Pact Broker also supports the "can I deploy?" query: before deploying a new version of a service, ask the broker whether all contracts with all counterparties are still satisfied. If not, the deployment is blocked.

### Alternatives to Pact

- **Spring Cloud Contract** — generates tests from contract definitions. More opinionated, Spring-ecosystem specific.
- **Schema Registry compatibility checks** — not exactly contract testing, but schema compatibility enforcement (backward, forward, full) provides a similar guarantee: new schemas won't break existing consumers.
- **AsyncAPI** — an OpenAPI-like specification for async APIs. Useful for documentation and code generation, less mature for contract testing.

---

## Consumer-Driven Contracts — Letting Consumers Define Expectations

Consumer-driven contracts (CDC) invert the traditional model. Instead of the producer defining "here's what I send" and consumers adapting, the consumer defines "here's what I need" and the producer verifies it can provide it.

This matters because in event-driven systems, a single topic might have dozens of consumers, each caring about different fields. The producer team doesn't necessarily know which fields each consumer depends on.

### How CDC Works in Practice

1. **Consumer team writes a contract:** "We (Shipping Service) need OrderPlaced events to contain `orderId`, `customerId`, and `shippingAddress` with at least `street`, `city`, and `zip`."

2. **Contract is shared with the producer** (via Pact Broker, git repo, or artifact repository).

3. **Producer's CI pipeline verifies** that the events it produces satisfy all consumer contracts.

4. **If a producer change breaks a contract,** the producer's build fails — not the consumer's. This is the critical difference. The team making the breaking change is the team that gets the failing test.

### The Governance Question

CDC only works if there's a process for managing contracts. Without governance, you get:

- Consumers adding contracts for fields that were never intended to be stable ("you renamed `customerId` to `customer_id` and broke us").
- An ever-growing set of contracts that prevents the producer from evolving ("we can't add the new field because consumer X's contract doesn't include it and their test fails on unknown fields").
- Contract proliferation where every consumer has a slightly different view of the same event.

The solution is a contract review process — producers and consumers agree on what constitutes the stable interface of an event, and contracts are written against that interface, not against the full event payload.

---

## Integration Testing with Embedded/In-Memory Brokers

Unit tests with mocked brokers verify logic. Integration tests with real (or real-enough) brokers verify that your code actually works when messages flow through infrastructure.

### Testcontainers

Testcontainers is the industry standard for integration testing with real infrastructure. It starts Docker containers for your tests and tears them down afterward. It supports Kafka, RabbitMQ, Pulsar, Redis, and essentially every broker you might use.

```java
// Java integration test with Testcontainers and Kafka
@Testcontainers
class OrderEventIntegrationTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
    ).withKraft();  // Use KRaft mode, no ZooKeeper needed

    private Producer<String, String> producer;
    private Consumer<String, String> consumer;

    @BeforeEach
    void setUp() {
        // Configure producer
        Properties producerProps = new Properties();
        producerProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,
                         kafka.getBootstrapServers());
        producerProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                         StringSerializer.class.getName());
        producerProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                         StringSerializer.class.getName());
        producer = new KafkaProducer<>(producerProps);

        // Configure consumer
        Properties consumerProps = new Properties();
        consumerProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,
                         kafka.getBootstrapServers());
        consumerProps.put(ConsumerConfig.GROUP_ID_CONFIG, "test-group");
        consumerProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        consumerProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                         StringDeserializer.class.getName());
        consumerProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                         StringDeserializer.class.getName());
        consumer = new KafkaConsumer<>(consumerProps);
    }

    @Test
    void orderPlacedEventFlowsThroughBroker() throws Exception {
        String topic = "orders-" + UUID.randomUUID(); // unique topic per test
        consumer.subscribe(Collections.singletonList(topic));

        // Produce an event
        String event = """
            {
                "eventType": "OrderPlaced",
                "orderId": "ord-123",
                "customerId": "cust-456"
            }
            """;
        producer.send(new ProducerRecord<>(topic, "ord-123", event)).get();

        // Consume and verify
        ConsumerRecords<String, String> records = ConsumerRecords.empty();
        Instant deadline = Instant.now().plusSeconds(10);

        while (records.isEmpty() && Instant.now().isBefore(deadline)) {
            records = consumer.poll(Duration.ofMillis(500));
        }

        assertFalse(records.isEmpty(), "Expected to receive the event");
        ConsumerRecord<String, String> record = records.iterator().next();
        assertEquals("ord-123", record.key());

        JsonNode eventNode = objectMapper.readTree(record.value());
        assertEquals("OrderPlaced", eventNode.get("eventType").asText());
    }

    @AfterEach
    void tearDown() {
        producer.close();
        consumer.close();
    }
}
```

```python
# Python integration test with testcontainers
import pytest
from testcontainers.kafka import KafkaContainer
from confluent_kafka import Producer, Consumer
import json
import time

@pytest.fixture(scope="module")
def kafka_container():
    with KafkaContainer(image="confluentinc/cp-kafka:7.5.0") as kafka:
        yield kafka

@pytest.fixture
def kafka_producer(kafka_container):
    producer = Producer({
        'bootstrap.servers': kafka_container.get_bootstrap_server(),
    })
    yield producer

@pytest.fixture
def kafka_consumer(kafka_container):
    consumer = Consumer({
        'bootstrap.servers': kafka_container.get_bootstrap_server(),
        'group.id': f'test-group-{uuid.uuid4()}',
        'auto.offset.reset': 'earliest',
    })
    yield consumer
    consumer.close()

def test_order_event_round_trip(kafka_producer, kafka_consumer):
    topic = f"orders-{uuid.uuid4()}"

    # Produce
    event = {
        "eventType": "OrderPlaced",
        "orderId": "ord-123",
        "customerId": "cust-456",
    }
    kafka_producer.produce(
        topic=topic,
        key="ord-123",
        value=json.dumps(event),
    )
    kafka_producer.flush()

    # Consume
    kafka_consumer.subscribe([topic])
    messages = []
    deadline = time.time() + 10

    while not messages and time.time() < deadline:
        msg = kafka_consumer.poll(timeout=0.5)
        if msg and not msg.error():
            messages.append(msg)

    assert len(messages) == 1
    received = json.loads(messages[0].value())
    assert received["eventType"] == "OrderPlaced"
    assert received["orderId"] == "ord-123"
```

### Testcontainers Tips and Warnings

- **Use unique topic names per test.** Reusing topic names across tests leads to test pollution, where one test's events leak into another test. A UUID suffix on the topic name solves this.
- **Use unique consumer group IDs per test.** Same reason. Consumer groups track offsets; shared groups mean shared state.
- **Set `auto.offset.reset=earliest`.** Otherwise your consumer might miss events that were produced before it subscribed.
- **Container startup time is real.** A Kafka container takes 10-30 seconds to start. Use `scope="module"` or `scope="session"` fixtures to share a container across tests.
- **Docker must be available.** This seems obvious until your CI pipeline doesn't have Docker. Testcontainers needs a Docker daemon. For CI, you might need Docker-in-Docker (DinD) or a remote Docker host.

### Embedded Brokers (When Docker Isn't Available)

Some brokers offer embeddable or in-memory versions for testing:

- **Embedded Kafka** (`spring-kafka-test` provides `EmbeddedKafka` for Spring Boot applications)
- **RabbitMQ** has `rabbitmq-mock` libraries
- **Redis** has embedded alternatives like `embedded-redis`
- **Pulsar** offers a standalone mode suitable for testing

Embedded brokers start faster than containers but may not behave identically to production brokers. They're a pragmatic choice when Docker isn't available but should not be your only level of integration testing.

---

## End-to-End Testing Strategies — The Timing Problem

End-to-end tests for event-driven systems verify that a complete flow works: an API call triggers an event, the event is consumed, a side effect occurs, and the final state is correct.

The fundamental challenge is the timing problem: when do you check for the expected outcome?

### The Naive Approach (Don't Do This)

```python
def test_order_flow_end_to_end():
    # Place an order via API
    response = requests.post("http://order-service/orders", json=order_data)
    assert response.status_code == 201
    order_id = response.json()["orderId"]

    # Wait for the event to be processed
    time.sleep(5)  # <-- THIS IS THE PROBLEM

    # Check that shipping was created
    shipping = requests.get(f"http://shipping-service/shipments?orderId={order_id}")
    assert shipping.status_code == 200
```

A fixed sleep is the most common approach and the worst. It's either too short (flaky test) or too long (slow test). Usually both, depending on the day.

### The Polling Approach (Better)

```python
def test_order_flow_end_to_end():
    response = requests.post("http://order-service/orders", json=order_data)
    order_id = response.json()["orderId"]

    # Poll until the expected state appears or timeout
    shipment = poll_until(
        fn=lambda: requests.get(
            f"http://shipping-service/shipments?orderId={order_id}"
        ),
        condition=lambda r: r.status_code == 200 and r.json().get("status") == "CREATED",
        timeout_seconds=30,
        interval_seconds=0.5,
    )

    assert shipment.json()["orderId"] == order_id


def poll_until(fn, condition, timeout_seconds, interval_seconds):
    """Poll a function until a condition is met or timeout expires."""
    deadline = time.time() + timeout_seconds
    last_result = None

    while time.time() < deadline:
        last_result = fn()
        if condition(last_result):
            return last_result
        time.sleep(interval_seconds)

    raise TimeoutError(
        f"Condition not met within {timeout_seconds}s. "
        f"Last result: {last_result}"
    )
```

Polling is better because it adapts to the actual processing time. When the system is fast, the test is fast. When the system is slow, the test waits longer (up to the timeout).

### The Event-Driven Approach (Best)

Instead of polling for the outcome, subscribe to an output event that signals completion:

```python
def test_order_flow_end_to_end():
    # Subscribe to the outcome event BEFORE triggering the flow
    outcome_consumer = create_consumer(topic="shipment-events")

    # Trigger the flow
    response = requests.post("http://order-service/orders", json=order_data)
    order_id = response.json()["orderId"]

    # Wait for the outcome event
    event = wait_for_event(
        consumer=outcome_consumer,
        predicate=lambda e: (
            e["eventType"] == "ShipmentCreated" and
            e["data"]["orderId"] == order_id
        ),
        timeout_seconds=30,
    )

    assert event["data"]["orderId"] == order_id
    assert event["data"]["status"] == "CREATED"
```

This is the most natural approach for event-driven systems — you're testing the system the way it actually works, by observing events.

---

## Testing Event Ordering and Idempotency

### Ordering Tests

If your system depends on event ordering (and most event-driven systems do, at least within a partition), you need tests that verify ordering is preserved.

```python
def test_events_processed_in_order():
    """Verify that events for the same key are processed in order."""
    order_id = f"ord-{uuid.uuid4()}"

    events = [
        {"eventType": "OrderPlaced", "orderId": order_id, "sequence": 1},
        {"eventType": "OrderPaid", "orderId": order_id, "sequence": 2},
        {"eventType": "OrderShipped", "orderId": order_id, "sequence": 3},
    ]

    # Produce all events with the same key (same partition)
    for event in events:
        producer.produce(
            topic="orders",
            key=order_id,
            value=json.dumps(event),
        )
    producer.flush()

    # Consume and verify ordering
    consumed = consume_n_events(consumer, n=3, timeout=15)
    consumed_sequences = [e["sequence"] for e in consumed]

    assert consumed_sequences == [1, 2, 3], \
        f"Events received out of order: {consumed_sequences}"
```

### Idempotency Tests

At-least-once delivery means your consumer might see the same event twice. Your tests should verify that processing an event twice produces the same result as processing it once.

```python
def test_order_placed_handler_is_idempotent():
    """Processing the same event twice should not create duplicate side effects."""
    handler = OrderPlacedHandler(
        inventory_service=real_inventory_service,
        db=test_database,
    )

    event = {
        "eventId": "evt-12345",
        "eventType": "OrderPlaced",
        "data": {
            "orderId": "ord-123",
            "items": [{"sku": "WIDGET-001", "quantity": 2}],
        }
    }

    # Process the event twice
    handler.handle(event)
    handler.handle(event)  # duplicate delivery

    # Verify the side effect happened exactly once
    reservations = test_database.query(
        "SELECT COUNT(*) FROM inventory_reservations WHERE order_id = %s",
        ("ord-123",)
    )
    assert reservations == 1, \
        f"Expected 1 reservation, got {reservations}. Handler is not idempotent."
```

> **Test the deduplication mechanism explicitly.** If your idempotency relies on an `eventId` stored in a deduplication table, write a test that verifies the deduplication table is populated after first processing and checked before second processing. Don't assume the mechanism works — prove it.

---

## Chaos Engineering: What Happens When the Broker Dies?

You've tested the happy path. Events flow, consumers process, state is correct. Now test what happens when the infrastructure misbehaves — because in production, it will.

### Toxiproxy: Network Chaos Made Easy

Toxiproxy sits between your services and the broker, injecting network faults on demand.

```python
# Toxiproxy setup for Kafka chaos testing
from toxiproxy import Toxiproxy

toxiproxy = Toxiproxy()

# Create a proxy in front of Kafka
kafka_proxy = toxiproxy.create(
    name="kafka",
    listen="0.0.0.0:19092",
    upstream="kafka-broker:9092"
)

# Your application connects through the proxy
producer_config = {
    'bootstrap.servers': 'toxiproxy:19092',  # proxy, not real broker
    # ... other config
}

def test_producer_retries_on_network_latency():
    """Producer should succeed even with network latency."""
    # Add 2 seconds of latency
    kafka_proxy.add_toxic(
        name="latency",
        type="latency",
        attributes={"latency": 2000, "jitter": 500}
    )

    try:
        # Producer should still succeed (with retries)
        future = producer.send("orders", event)
        result = future.get(timeout=30)
        assert result.offset is not None
    finally:
        kafka_proxy.remove_toxic("latency")


def test_consumer_recovers_from_broker_disconnect():
    """Consumer should resume processing after a broker outage."""
    # Produce some events
    for i in range(10):
        producer.produce("orders", f'{{"seq": {i}}}')
    producer.flush()

    # Cut the connection
    kafka_proxy.add_toxic(
        name="disconnect",
        type="reset_peer",
        attributes={"timeout": 0}
    )

    # Wait for the outage to be noticed
    time.sleep(5)

    # Restore the connection
    kafka_proxy.remove_toxic("disconnect")

    # Verify the consumer catches up
    consumed = consume_all_events(consumer, topic="orders", timeout=30)
    assert len(consumed) == 10, f"Expected 10 events, got {len(consumed)}"
```

### Chaos Scenarios to Test

| Scenario | What You're Testing | How to Inject |
|----------|-------------------|---------------|
| Broker becomes unreachable | Producer retries, consumer reconnection | Toxiproxy `reset_peer` or Docker `pause` |
| High network latency | Timeout handling, request timeouts | Toxiproxy `latency` toxic |
| Packet loss | Retry logic, duplicate handling | Toxiproxy `bandwidth` toxic with limit |
| Broker disk full | Producer backpressure, error handling | Fill the container's disk |
| Consumer process crash mid-processing | Offset management, reprocessing | `kill -9` the consumer process |
| Rebalancing during processing | At-least-once processing, offset commit timing | Add/remove consumers from the group |
| Schema registry unavailable | Serialization failure handling | Stop the schema registry container |

### Load Testing Event Pipelines

Event-driven systems have different failure modes under load than synchronous systems. Instead of returning HTTP 503, they silently accumulate lag. A system that looks fine at 1,000 events/second might fall apart at 10,000 — but the failure manifests as growing consumer lag, not immediate errors.

```python
# Simple load test for event throughput
import time
from confluent_kafka import Producer
import json

def load_test_producer(bootstrap_servers, topic, num_events, batch_size=1000):
    producer = Producer({
        'bootstrap.servers': bootstrap_servers,
        'linger.ms': 50,          # batch events for better throughput
        'batch.num.messages': 10000,
        'queue.buffering.max.messages': 100000,
        'compression.type': 'lz4',
    })

    start = time.time()
    delivered = 0
    errors = 0

    def delivery_callback(err, msg):
        nonlocal delivered, errors
        if err:
            errors += 1
        else:
            delivered += 1

    for i in range(num_events):
        event = {
            "eventType": "LoadTestEvent",
            "sequence": i,
            "timestamp": time.time(),
            "payload": "x" * 500,  # ~500 byte payload
        }
        producer.produce(
            topic=topic,
            key=str(i % 100),  # distribute across 100 keys
            value=json.dumps(event),
            callback=delivery_callback,
        )

        # Periodically flush to avoid buffer overflow
        if i % batch_size == 0:
            producer.flush()

    producer.flush()
    elapsed = time.time() - start

    print(f"Produced {num_events} events in {elapsed:.2f}s")
    print(f"Throughput: {num_events / elapsed:.0f} events/sec")
    print(f"Delivered: {delivered}, Errors: {errors}")

# Run it
load_test_producer(
    bootstrap_servers="broker1:9092",
    topic="load-test",
    num_events=1_000_000
)
```

What to measure during a load test:

- **Producer throughput** (events/second)
- **Consumer throughput** (events/second consumed)
- **Consumer lag** (the gap between what's been produced and what's been consumed)
- **End-to-end latency** (time from produce to consume — embed a timestamp in the event)
- **Broker resource utilization** (CPU, memory, disk I/O, network)
- **Error rates** (serialization failures, timeout errors, rebalancing events)

The most important metric is consumer lag under sustained load. If lag grows unboundedly, your consumers can't keep up, and you need to either add consumers, optimize processing, or increase parallelism.

---

## Testing Schema Evolution

Schema evolution — changing the format of events over time — is inevitable. Testing that old consumers can handle new schemas (backward compatibility) and new consumers can handle old schemas (forward compatibility) prevents production outages during deployment.

```python
# Test backward compatibility: new schema, old consumer
def test_old_consumer_handles_new_event_format():
    """An existing consumer should gracefully handle events with new fields."""

    # Old event format (v1)
    v1_event = {
        "eventType": "OrderPlaced",
        "version": 1,
        "data": {
            "orderId": "ord-123",
            "customerId": "cust-456",
            "totalAmount": "59.98",
        }
    }

    # New event format (v2) — added 'currency' and 'loyaltyPoints'
    v2_event = {
        "eventType": "OrderPlaced",
        "version": 2,
        "data": {
            "orderId": "ord-456",
            "customerId": "cust-789",
            "totalAmount": "99.99",
            "currency": "EUR",       # new field
            "loyaltyPoints": 150,    # new field
        }
    }

    # The v1 consumer should handle both
    consumer = OrderPlacedConsumerV1()

    consumer.handle(v1_event)   # should work (same version)
    consumer.handle(v2_event)   # should work (ignores unknown fields)

    # Verify both were processed correctly
    assert len(consumer.processed_orders) == 2


# Test forward compatibility: old schema, new consumer
def test_new_consumer_handles_old_event_format():
    """A new consumer should handle events from before the schema change."""

    v1_event = {
        "eventType": "OrderPlaced",
        "version": 1,
        "data": {
            "orderId": "ord-123",
            "customerId": "cust-456",
            "totalAmount": "59.98",
            # no 'currency' field — it didn't exist in v1
        }
    }

    consumer = OrderPlacedConsumerV2()  # expects currency but has a default
    consumer.handle(v1_event)

    order = consumer.processed_orders[0]
    assert order.total_amount == Decimal("59.98")
    assert order.currency == "USD"  # default value when field is absent
```

### Schema Registry Compatibility Testing

If you're using a schema registry (you should be), test compatibility before registering a new schema:

```bash
#!/bin/bash
# Test schema compatibility before deployment

SCHEMA_REGISTRY_URL="http://schema-registry:8081"
SUBJECT="orders-value"
NEW_SCHEMA_FILE="schemas/order-placed-v2.avsc"

# Check compatibility
RESULT=$(curl -s -X POST \
  "${SCHEMA_REGISTRY_URL}/compatibility/subjects/${SUBJECT}/versions/latest" \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d "{\"schema\": $(cat ${NEW_SCHEMA_FILE} | jq -Rs .)}")

IS_COMPATIBLE=$(echo $RESULT | jq -r '.is_compatible')

if [ "$IS_COMPATIBLE" != "true" ]; then
  echo "INCOMPATIBLE SCHEMA CHANGE DETECTED"
  echo "Details: $RESULT"
  exit 1
fi

echo "Schema is compatible. Safe to deploy."
```

---

## The Test Pyramid for Event-Driven Systems

The traditional test pyramid (many unit tests, fewer integration tests, even fewer E2E tests) doesn't map cleanly onto event-driven systems. A more accurate model:

```
                    /\
                   /  \
                  / E2E \          (few, slow, high-confidence)
                 /--------\
                /  Chaos    \      (periodic, infrastructure-focused)
               /  Engineering \
              /----------------\
             / Contract Tests    \  (per-consumer, per-producer)
            /--------------------\
           /  Integration Tests    \  (with real broker via Testcontainers)
          /------------------------\
         /  Unit Tests (handlers)    \  (fast, many, logic-focused)
        /----------------------------\
```

- **Unit tests:** Test event construction, handler logic, serialization/deserialization. Fast. Many. No broker.
- **Integration tests:** Test event flow through a real broker. Producer -> broker -> consumer. Testcontainers. Slower. Fewer.
- **Contract tests:** Verify producer-consumer agreements. Can be run independently. Medium speed.
- **Chaos tests:** Verify resilience. Periodic, not on every commit. Slow.
- **E2E tests:** Verify complete business flows. Few. Slow. High-maintenance. Essential.

### Coverage Guidance

| What to Test | Level | How |
|-------------|-------|-----|
| Event payload construction | Unit | Mock the publisher |
| Event handler business logic | Unit | Pass in events directly |
| Serialization/deserialization | Unit | Round-trip test with schema |
| Event flow through broker | Integration | Testcontainers |
| Producer-consumer contract | Contract | Pact or schema compatibility |
| Ordering guarantees | Integration | Produce N events, verify order |
| Idempotency | Integration | Process same event twice, verify state |
| Error handling (poison pill) | Integration | Produce invalid event, verify DLQ |
| Schema evolution | Integration + Contract | Both old and new formats |
| Broker failure recovery | Chaos | Toxiproxy, container stop/start |
| Consumer lag under load | Load | Sustained traffic test |
| Complete business flow | E2E | API to final state, polling or event-based |

---

## Anti-Patterns: Testing the Broker and Flaky Async Assertions

### Anti-Pattern 1: Testing the Broker

```python
# DON'T DO THIS
def test_kafka_retains_messages():
    """Verify Kafka retains messages for the configured retention period."""
    # ...
    # This is Kafka's job. Kafka has its own tests. Test YOUR code.
```

You're not testing Kafka. You're not testing RabbitMQ. You didn't write them. They have their own test suites. Test your code's interaction with the broker, not the broker's behavior.

### Anti-Pattern 2: Fixed Sleep in Assertions

```python
# DON'T DO THIS
def test_event_is_consumed():
    producer.produce("orders", event)
    time.sleep(10)  # hope and pray
    assert consumer.last_event == event
```

Use polling with a timeout. Use event-based verification. Use Awaitility (Java) or tenacity (Python). Never use a fixed sleep as your synchronization mechanism.

### Anti-Pattern 3: Shared State Between Tests

```python
# DON'T DO THIS
TOPIC = "orders"  # all tests share this topic

def test_order_placed():
    produce_to(TOPIC, order_placed_event)
    # might consume an event from a different test
```

Use unique topic names per test. Use unique consumer groups per test. Tests should be independent.

### Anti-Pattern 4: Testing Only the Happy Path

If your test suite doesn't include tests for: malformed events, duplicate events, events arriving out of order, broker unavailability, and schema mismatches — your test suite is a wish list, not a verification.

### Anti-Pattern 5: Not Testing Consumer Offset Management

```python
# DO THIS — verify your consumer resumes correctly after restart
def test_consumer_resumes_from_last_committed_offset():
    """After restart, consumer should process only new events."""
    # Produce 5 events
    for i in range(5):
        producer.produce("orders", f'{{"seq": {i}}}')
    producer.flush()

    # Consume all 5 and commit
    consumed_first = consume_n_events(consumer, n=5, timeout=10)
    consumer.commit()
    consumer.close()

    # Produce 3 more events
    for i in range(5, 8):
        producer.produce("orders", f'{{"seq": {i}}}')
    producer.flush()

    # Create a new consumer with the same group ID
    new_consumer = create_consumer(group_id=consumer_group_id)
    consumed_second = consume_n_events(new_consumer, n=3, timeout=10)

    # Should get events 5, 6, 7 — not 0-4 again
    sequences = [json.loads(e)["seq"] for e in consumed_second]
    assert sequences == [5, 6, 7]
```

---

## Summary

Testing event-driven systems requires a fundamentally different approach from testing synchronous systems. The key principles:

1. **Separate business logic from infrastructure.** Extract event handlers into testable units that don't depend on the broker.
2. **Use contract tests** to verify producer-consumer agreements without running both simultaneously.
3. **Use Testcontainers** for integration tests with real brokers. Embedded fakes are acceptable fallbacks, not first choices.
4. **Never use fixed sleeps.** Use polling, event-based verification, or awaiting libraries.
5. **Test idempotency explicitly.** Process every event at least twice in your tests.
6. **Test failure modes.** Chaos engineering isn't optional for production event-driven systems.
7. **Test schema evolution.** Old consumers with new events. New consumers with old events. Both directions.
8. **Use unique topics and consumer groups per test.** Shared state between tests is the fastest path to a flaky test suite.

The goal isn't 100% coverage — it's confidence that your system behaves correctly under normal conditions and degrades gracefully under abnormal ones. In an event-driven system, "abnormal conditions" includes most of the conditions you'll encounter in production.
