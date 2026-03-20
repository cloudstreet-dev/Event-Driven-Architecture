# Schema Evolution and Contracts

You've got your events flowing, your consumers humming, your dashboards green. Life is good. Then someone adds a field to an event, and three services fall over at 2 AM on a Saturday. Welcome to schema evolution — the problem that everyone acknowledges is important and nobody budgets time for until it's too late.

Schema evolution is the discipline of changing what your events look like over time without setting fire to every consumer downstream. It sounds straightforward. It is not. It is, in fact, the hardest problem you will underestimate in an event-driven architecture, because it sits at the intersection of technical constraints, organizational politics, and the fundamental human inability to predict the future.

---

## Why Schema Evolution Is the Hardest Problem You'll Underestimate

In a monolithic application, changing a data structure is a compile-time problem. You change the struct, the compiler screams at you, you fix the fifteen call sites, and you go home. In an event-driven system, changing an event schema is a *deployment-time* problem spread across multiple teams, multiple services, and multiple time zones.

Here's what makes it genuinely difficult:

1. **Producers and consumers deploy independently.** You cannot coordinate a simultaneous upgrade across all services that touch an event. You will have old producers running alongside new consumers, and new producers running alongside old consumers, sometimes for days, sometimes for months, sometimes — let's be honest — forever.

2. **Events are durable.** Unlike HTTP request/response payloads that exist for milliseconds, events sit in logs and queues. A Kafka topic with a 30-day retention period means your new consumer must handle events written by code that was deployed a month ago. Possibly code written by an engineer who has since left the company.

3. **The blast radius is invisible.** When you change an HTTP API, you can look at your API gateway logs and enumerate your callers. When you change an event schema, you may not even know who's consuming it. That analytics team that tapped into your order events six months ago? They didn't tell you. They're about to have a very bad day.

4. **Serialization formats have opinions.** Your choice of serialization format determines what kinds of changes are safe, what kinds are dangerous, and what kinds are impossible. Choose wrong early, pay forever.

The net result is that schema evolution requires a level of discipline and tooling that feels disproportionate to the apparent simplicity of "I just want to add a field." But the alternative — no discipline, no tooling — is a system that becomes progressively more terrifying to change.

---

## Schema Formats: The Serialization Wars

Before we talk about evolving schemas, we need to talk about what schemas *are*, because the format you choose constrains everything that follows.

### JSON (with JSON Schema)

JSON is the lingua franca of web APIs, and plenty of event-driven systems use it too. It's human-readable, self-describing (sort of), universally supported, and — this is the critical part — **has no built-in schema enforcement**.

JSON Schema exists as a specification for describing the shape of JSON documents, but it's a validation layer bolted on after the fact, not a feature of the format itself. This means:

- **Producers can emit whatever they want.** There's no serialization step that rejects invalid events. You need explicit validation.
- **Consumers must be defensive.** You cannot trust that a field exists, has the right type, or means what you think it means.
- **Schema evolution is "easy" in the worst sense.** Anyone can add, remove, or change fields at any time. The format won't stop them. The 3 AM page will.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "orderId": { "type": "string", "format": "uuid" },
    "customerId": { "type": "string" },
    "totalAmount": { "type": "number", "minimum": 0 },
    "currency": { "type": "string", "enum": ["USD", "EUR", "GBP"] },
    "createdAt": { "type": "string", "format": "date-time" }
  },
  "required": ["orderId", "customerId", "totalAmount", "currency", "createdAt"],
  "additionalProperties": false
}
```

That `additionalProperties: false` at the bottom? It's the source of a religious war. Set it to `false` and you get strict validation but break forward compatibility (consumers reject events with new fields). Set it to `true` (or omit it) and you get forward compatibility but lose the ability to catch typos and garbage fields.

**Verdict:** JSON is fine for systems with a small number of well-coordinated teams. It becomes increasingly painful as the number of independent producers and consumers grows. The lack of a built-in schema enforcement mechanism means you're relying on convention and discipline, which — let's be charitable — have a mixed track record.

### Apache Avro

Avro is the schema evolution format. It was designed for exactly this problem, and it shows. An Avro schema defines the structure of your data, and the serialization/deserialization process is schema-aware: the writer's schema and the reader's schema are both available at read time, and the Avro library resolves differences between them.

```json
{
  "type": "record",
  "name": "OrderCreated",
  "namespace": "com.example.orders",
  "fields": [
    { "name": "orderId", "type": "string" },
    { "name": "customerId", "type": "string" },
    { "name": "totalAmount", "type": "double" },
    { "name": "currency", "type": "string", "default": "USD" },
    { "name": "createdAt", "type": "long", "logicalType": "timestamp-millis" },
    {
      "name": "discountCode",
      "type": ["null", "string"],
      "default": null
    }
  ]
}
```

Key Avro features for schema evolution:

- **Default values** make adding fields backward-compatible. Old data missing the new field gets the default.
- **Union types** (like `["null", "string"]`) let you make fields optional without contortions.
- **The reader uses its own schema**, so it only sees the fields it cares about, even if the writer included extras.
- **Schema resolution rules** are explicit and well-defined. You don't have to guess what happens when schemas diverge.

The cost? Avro requires both the writer's and reader's schemas at deserialization time. In practice, this means you need a schema registry (more on that shortly). Also, Avro's binary encoding is not human-readable, which makes debugging harder — you can't just `cat` a message and see what's in it.

**Verdict:** If schema evolution is a first-class concern (and it should be), Avro is the strongest choice. The Confluent ecosystem is built around it for good reason.

### Protocol Buffers (Protobuf)

Google's Protocol Buffers take a different approach. Schemas are defined in `.proto` files and compiled into language-specific code. Every field has a numeric tag, and the wire format uses these tags rather than field names.

```protobuf
syntax = "proto3";

package orders;

message OrderCreated {
  string order_id = 1;
  string customer_id = 2;
  double total_amount = 3;
  string currency = 4;
  int64 created_at = 5;

  // Added in v2 — old consumers will ignore this field
  optional string discount_code = 6;
}
```

Protobuf's evolution model:

- **Adding fields is safe** as long as you use new tag numbers. Old readers ignore unknown tags.
- **Removing fields is safe** as long as you never reuse the tag number. (The `reserved` keyword helps enforce this.)
- **Renaming fields is free** because the wire format uses tag numbers, not names. This is either a feature or a footgun depending on your perspective.
- **Changing field types is dangerous.** Some type changes are compatible (e.g., `int32` to `int64`), but most are not.

```protobuf
// DANGER: reserved tags and names prevent accidental reuse
message OrderCreated {
  reserved 7, 8;           // These tag numbers are retired
  reserved "legacy_field"; // This name is retired

  string order_id = 1;
  // ... rest of fields
}
```

The `proto3` syntax removed required fields entirely (everything is implicitly optional with a zero/empty default), which simplifies evolution but makes it harder to express "this field must be present" — a constraint you then have to enforce in application code.

**Verdict:** Protobuf is excellent for evolution, has superb cross-language support, and produces compact wire formats. The tag-based approach is fundamentally sound. The main friction is the code generation step, which some teams find annoying and others find essential.

### Apache Thrift

Thrift, originally from Facebook, is similar to Protobuf in concept: an IDL that compiles to language-specific code with tagged fields on the wire. It supports required, optional, and default fields.

```thrift
struct OrderCreated {
  1: required string orderId,
  2: required string customerId,
  3: required double totalAmount,
  4: optional string currency = "USD",
  5: required i64 createdAt,
  6: optional string discountCode
}
```

Thrift's evolution rules mirror Protobuf's: new fields with new IDs are safe, removing optional fields is safe, never reuse field IDs. The `required` keyword is a trap — once a field is required, you can never remove it without breaking existing readers, which is why Protobuf dropped the concept entirely in proto3.

**Verdict:** Thrift is a perfectly serviceable choice, but it's lost mindshare to Protobuf. Unless you're already invested in the Thrift ecosystem, there's little reason to choose it for new projects.

### The Comparison, Honestly

| Concern | JSON Schema | Avro | Protobuf | Thrift |
|---|---|---|---|---|
| Human readability | Excellent | Poor (binary) | Poor (binary) | Poor (binary) |
| Schema enforcement | Opt-in | Built-in | Built-in (codegen) | Built-in (codegen) |
| Evolution rules | Ad hoc | Formal, well-defined | Formal, tag-based | Formal, tag-based |
| Schema registry support | Varies | Excellent (Confluent) | Good | Limited |
| Language support | Universal | Good (JVM-centric) | Excellent | Good |
| Wire format size | Large | Compact | Very compact | Compact |
| Debugging ease | Easy | Hard | Hard | Hard |

---

## Compatibility Types: A Precise Vocabulary

When we say a schema change is "compatible," we need to be specific about *which direction* the compatibility runs. There are three types, and conflating them is a reliable source of production incidents.

### Backward Compatibility

A new schema is **backward compatible** if it can read data written with the old schema.

This is the most common requirement. Your consumers upgrade to the new schema, and they can still process events that were written before the upgrade.

Safe changes under backward compatibility:
- **Adding a field with a default value.** Old events don't have it; the default fills in.
- **Removing a field.** New readers just ignore it. (But old readers might still need it — see forward compatibility.)

```json
// Schema v1
{
  "type": "record",
  "name": "OrderCreated",
  "fields": [
    { "name": "orderId", "type": "string" },
    { "name": "totalAmount", "type": "double" }
  ]
}

// Schema v2 — BACKWARD COMPATIBLE (added field with default)
{
  "type": "record",
  "name": "OrderCreated",
  "fields": [
    { "name": "orderId", "type": "string" },
    { "name": "totalAmount", "type": "double" },
    { "name": "currency", "type": "string", "default": "USD" }
  ]
}
```

A v2 reader processing a v1 event will see `currency` as `"USD"`. No crash, no null pointer, no existential crisis.

### Forward Compatibility

A new schema is **forward compatible** if data written with the new schema can be read by old consumers.

This is what you need when producers upgrade before consumers — which, in a microservices world, is roughly always. Your payment service ships a new version of `OrderCreated` on Tuesday, but the fulfillment service won't deploy until Thursday. Forward compatibility means Thursday isn't a disaster.

Safe changes under forward compatibility:
- **Adding a field.** Old readers ignore fields they don't know about (assuming the format supports this — Avro and Protobuf do, strict JSON Schema does not).
- **Removing a field that has a default.** Old readers expecting the field get the default.

### Full Compatibility

**Full compatibility** means the schema is both backward and forward compatible. This is the gold standard and the hardest to maintain.

Safe changes under full compatibility:
- **Adding a field with a default value.** New readers use the default for old events; old readers ignore the new field.

That's... basically it. Full compatibility is restrictive by design. It means:
- You cannot add required fields (breaks backward compatibility).
- You cannot remove fields without defaults (breaks forward compatibility).
- You cannot change field types.
- You cannot rename fields (in name-based formats).

This sounds limiting, and it is. It's also the only level that lets producers and consumers upgrade in any order without coordination. The constraint is the feature.

### Transitive Compatibility

There's one more dimension: **transitive** compatibility means the new schema is compatible not just with the immediately previous version, but with *all* previous versions.

Non-transitive: v3 is compatible with v2 (but maybe not v1).
Transitive: v3 is compatible with v2 AND v1 AND every version before that.

You want transitive compatibility. You almost certainly want transitive compatibility. If your Kafka topic has a 30-day retention period and you've released three schema versions this month, consumers processing old events need compatibility all the way back, not just one version.

---

## Schema Registries: The Adult Supervision Your Events Need

A schema registry is a centralized service that stores and manages event schemas. It is not optional. I mean, technically it's optional in the same way that seatbelts are optional — you can absolutely drive without one, and everything will be fine right up until it isn't.

### What a Schema Registry Does

1. **Stores schemas** with version history. Every version of every event schema, forever (or until you clean up, which — spoiler — you won't).
2. **Assigns schema IDs.** Each schema version gets a unique numeric ID. Producers embed this ID in the event payload so consumers know which schema to use for deserialization.
3. **Enforces compatibility.** This is the killer feature. When a producer tries to register a new schema version, the registry checks it against previous versions and *rejects it if it violates the compatibility rules*. This is your safety net. This is the thing that prevents the 2 AM page.
4. **Provides schema lookup.** Consumers fetch schemas by ID, so they can deserialize events without needing the schema baked into their code at compile time.

### The Major Registries

#### Confluent Schema Registry

The 800-pound gorilla. Tightly integrated with the Kafka ecosystem, supports Avro, Protobuf, and JSON Schema. Provides compatibility checking at the subject level (where a subject typically maps to a topic + record type).

```bash
# Register a schema
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\":\"record\",\"name\":\"OrderCreated\",\"fields\":[{\"name\":\"orderId\",\"type\":\"string\"}]}"}' \
  http://localhost:8081/subjects/orders-value/versions

# Check compatibility before registering
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\":\"record\",\"name\":\"OrderCreated\",\"fields\":[{\"name\":\"orderId\",\"type\":\"string\"},{\"name\":\"currency\",\"type\":\"string\",\"default\":\"USD\"}]}"}' \
  http://localhost:8081/compatibility/subjects/orders-value/versions/latest
```

The compatibility levels it supports:
- `BACKWARD` (default): new schema can read old data
- `FORWARD`: old schema can read new data
- `FULL`: both directions
- `BACKWARD_TRANSITIVE`, `FORWARD_TRANSITIVE`, `FULL_TRANSITIVE`: same, but against all versions
- `NONE`: no checking (the "I like to live dangerously" setting)

**Licensing note:** The Confluent Schema Registry is under the Confluent Community License, not Apache 2.0. This matters if you're building a competing SaaS product. For most internal use cases, it's fine.

#### Apicurio Registry

The open-source alternative, Apache 2.0 licensed. Supports Avro, Protobuf, JSON Schema, GraphQL, OpenAPI, and more. It can use Kafka, SQL databases, or in-memory storage as its backend.

Apicurio provides compatibility checking and supports the same compatibility levels as Confluent. It also offers a REST API that's mostly compatible with Confluent's, so migration between the two is not catastrophic.

**When to use it:** When you want open-source licensing, when you need support for non-Kafka brokers, or when you need to store non-event schemas (like OpenAPI specs) in the same registry.

#### AWS Glue Schema Registry

Amazon's managed offering. Integrates with Kinesis, MSK, and Lambda. Supports Avro and JSON Schema (Protobuf support was added later and with caveats).

**When to use it:** When you're all-in on AWS and want managed infrastructure with IAM integration. The compatibility checking is solid. The ecosystem integration is convenient. The vendor lock-in is real.

### Schema ID Embedding

In practice, the producer serializes an event like this:

```
[Magic Byte (1)][Schema ID (4 bytes)][Avro Payload (N bytes)]
```

The magic byte (`0x00`) signals that this is a schema-registry-aware payload. The 4-byte schema ID tells the consumer exactly which schema to fetch for deserialization. This is the Confluent wire format, and it's become a de facto standard even outside the Confluent ecosystem.

```python
# Python example with confluent-kafka and Avro
from confluent_kafka.avro import AvroProducer

producer = AvroProducer({
    'bootstrap.servers': 'localhost:9092',
    'schema.registry.url': 'http://localhost:8081'
}, default_value_schema=order_created_schema)

producer.produce(
    topic='orders',
    value={
        'orderId': 'ord-123',
        'customerId': 'cust-456',
        'totalAmount': 99.99,
        'currency': 'USD',
        'createdAt': 1679616000000
    }
)
producer.flush()
```

The producer doesn't manually embed the schema ID — the `AvroProducer` handles registration and embedding transparently. This is the happy path. The unhappy path is when someone bypasses the serializer and writes raw JSON to an Avro topic, which will be detected approximately never and cause havoc approximately always.

---

## Versioning Strategies

So you need to change an event schema. How do you manage the transition? There are several strategies, each with different tradeoffs between safety, complexity, and how much you trust your fellow engineers.

### Additive-Only Changes (The Golden Rule)

The simplest and most robust strategy: **you only ever add new optional fields with default values. You never remove fields. You never change field types. You never rename things.**

```json
// v1
{ "orderId": "string", "totalAmount": "double" }

// v2 — added optional field
{ "orderId": "string", "totalAmount": "double", "currency": "string (default: USD)" }

// v3 — added another optional field
{ "orderId": "string", "totalAmount": "double", "currency": "string (default: USD)", "discountPercent": "double (default: 0.0)" }
```

This is boring. This is predictable. This works. Every version is fully compatible with every other version, transitively. Consumers can upgrade at their leisure. Producers can upgrade without coordination.

The downside is that your schema accumulates fields over time. That `legacyCustomerType` field that was added in 2019 and hasn't been populated since 2021? It's still there. It will always be there. You're building geological strata in your event schemas.

### Semantic Versioning for Events

Borrowing from library versioning: **MAJOR.MINOR.PATCH** for event schemas.

- **PATCH** (1.0.0 -> 1.0.1): Documentation changes, no schema change.
- **MINOR** (1.0.0 -> 1.1.0): Backward-compatible additions (new optional fields).
- **MAJOR** (1.0.0 -> 2.0.0): Breaking changes.

The version can live in the event metadata:

```json
{
  "metadata": {
    "eventType": "OrderCreated",
    "schemaVersion": "2.1.0",
    "timestamp": "2025-03-15T10:30:00Z"
  },
  "payload": {
    "orderId": "ord-123",
    "customerId": "cust-456",
    "totalAmount": 99.99
  }
}
```

This gives consumers information to route or reject events. A consumer that handles `OrderCreated` v1.x can skip v2.x events (or send them to a dead letter queue) rather than crash trying to parse them.

**The catch:** Semantic versioning requires discipline and agreement on what constitutes a breaking change. You'd be surprised how many teams debate whether adding a new enum value is a minor or major change. (It depends on the serialization format: in Avro, it can be breaking if the reader has a fixed enum. In Protobuf, it's fine. In JSON, it depends on whether consumers validate enums.)

### The "v2 Topic" Approach

When you have a truly breaking change, one brutal-but-effective strategy is to create a new topic:

```
orders.order-created.v1  →  the old events
orders.order-created.v2  →  the new events
```

The v1 topic continues to receive events from old producers and serve old consumers. The v2 topic starts receiving events from upgraded producers. Over time, you migrate consumers from v1 to v2 and eventually decommission v1.

**Advantages:**
- Clean separation. No compatibility gymnastics.
- Old and new consumers can coexist indefinitely.
- Easy to reason about — each topic has exactly one schema.

**Disadvantages:**
- You need a migration period where both topics are active.
- Producers might need to dual-write during the transition (write to both v1 and v2).
- Consumers need to be updated to read from the new topic, which requires coordination — the very thing you were trying to avoid.
- Topic proliferation. After a few years, you have `v1`, `v2`, `v3`, and nobody's sure if `v1` is still active.

```python
# Dual-writing during migration period
class OrderEventProducer:
    def __init__(self, producer, migration_active=True):
        self.producer = producer
        self.migration_active = migration_active

    def publish_order_created(self, order):
        # Always write to v2
        v2_event = self._to_v2_schema(order)
        self.producer.produce('orders.order-created.v2', v2_event)

        # During migration, also write to v1 for lagging consumers
        if self.migration_active:
            v1_event = self._to_v1_schema(order)
            self.producer.produce('orders.order-created.v1', v1_event)

    def _to_v1_schema(self, order):
        return {'orderId': order.id, 'totalAmount': order.total}

    def _to_v2_schema(self, order):
        return {
            'orderId': order.id,
            'totalAmount': order.total,
            'currency': order.currency,
            'lineItems': [item.to_dict() for item in order.items]
        }
```

### Event Upcasting

A middle ground: keep a single topic but transform old events to the new schema at read time. The consumer maintains "upcasters" that know how to convert from old schema versions to the current one.

```java
public class OrderCreatedUpcaster {

    public OrderCreatedV3 upcast(JsonNode event, int schemaVersion) {
        return switch (schemaVersion) {
            case 1 -> upcastFromV1(event);
            case 2 -> upcastFromV2(event);
            case 3 -> parseV3(event);
            default -> throw new UnknownSchemaVersionException(schemaVersion);
        };
    }

    private OrderCreatedV3 upcastFromV1(JsonNode event) {
        return OrderCreatedV3.builder()
            .orderId(event.get("orderId").asText())
            .totalAmount(event.get("totalAmount").asDouble())
            .currency("USD")  // v1 didn't have currency, assume USD
            .lineItems(List.of())  // v1 didn't have line items
            .build();
    }

    private OrderCreatedV3 upcastFromV2(JsonNode event) {
        return OrderCreatedV3.builder()
            .orderId(event.get("orderId").asText())
            .totalAmount(event.get("totalAmount").asDouble())
            .currency(event.get("currency").asText())
            .lineItems(List.of())  // v2 didn't have line items
            .build();
    }
}
```

This keeps the data layer simple (one topic, old events stay as-is) at the cost of application complexity (every consumer needs upcasting logic). It works well when you have a small number of consumers and can keep the upcasters in a shared library.

---

## Breaking Changes and Migration Strategies

Sometimes you need to make a genuinely breaking change. The field type needs to change from a string to a structured object. The event needs to be split into two events. The semantics are changing fundamentally.

### The Expand-and-Contract Pattern

Borrowed from database migrations, this is the safest approach for most breaking changes:

**Phase 1: Expand.** Add the new field alongside the old one. Producers populate both. This is a backward-compatible change.

```json
// Phase 1: Both fields present
{
  "orderId": "ord-123",
  "customerName": "Jane Doe",
  "customer": {
    "id": "cust-456",
    "name": "Jane Doe",
    "email": "jane@example.com"
  }
}
```

**Phase 2: Migrate.** Update all consumers to read from the new field. Verify (through metrics and monitoring) that no consumer is still reading the old field.

**Phase 3: Contract.** Remove the old field. Producers stop populating it.

The total time for this process is "however long it takes to get every consumer team to update their code," which in practice ranges from "two weeks" to "we gave up and the old field is still there."

### The Event Splitter

When one event needs to become two, use a splitter service:

```
[Producer] → OrderCreated → [Splitter] → OrderCreated (slim)
                                       → OrderLineItemsCreated
```

The splitter consumes the old combined event and publishes two new events. Old consumers continue reading the old topic. New consumers read the new topics. The splitter runs until all consumers have migrated.

### The Nuclear Option: Replay with Transform

If you need to transform the entire history of events, you can:

1. Create a new topic with the new schema.
2. Write a batch job that reads every event from the old topic, transforms it, and writes it to the new topic.
3. Migrate consumers to the new topic.
4. Decommission the old topic.

This is expensive, disruptive, and sometimes the only option. It's the schema evolution equivalent of "we're going to need a bigger boat."

---

## Consumer-Driven Contracts

Here's an uncomfortable truth: the producer doesn't know what the consumer needs. The producer publishes what it has, and hopes it's enough. Consumer-driven contracts flip this: consumers declare what they need, and the system verifies that the producer provides it.

### The Concept

Each consumer publishes a **contract** describing the minimum set of fields and types it requires from an event. These contracts are tested against the producer's schema in CI/CD, and a failure blocks deployment.

```json
// Consumer contract: fulfillment-service needs from OrderCreated
{
  "consumer": "fulfillment-service",
  "event": "OrderCreated",
  "requiredFields": {
    "orderId": "string",
    "customerId": "string",
    "shippingAddress": {
      "street": "string",
      "city": "string",
      "postalCode": "string",
      "country": "string"
    }
  }
}
```

### Implementing Consumer-Driven Contracts

The tooling for this in the event-driven world is less mature than for HTTP APIs (where [Pact](https://pact.io/) is the standard bearer), but the principle is the same:

1. **Consumers define contracts** specifying what fields they read and what types they expect.
2. **Contracts are stored** in a shared repository or the schema registry.
3. **CI/CD pipelines** verify that the producer's schema satisfies all consumer contracts before deploying.
4. **A producer cannot make a change** that violates any consumer's contract.

```python
# Simplified contract verification
def verify_contract(producer_schema: dict, consumer_contract: dict) -> list:
    violations = []

    for field_name, expected_type in consumer_contract['requiredFields'].items():
        producer_field = find_field(producer_schema, field_name)

        if producer_field is None:
            violations.append(f"Missing required field: {field_name}")
        elif not types_compatible(producer_field['type'], expected_type):
            violations.append(
                f"Type mismatch for {field_name}: "
                f"producer has {producer_field['type']}, "
                f"consumer expects {expected_type}"
            )

    return violations
```

### The Organizational Challenge

Consumer-driven contracts sound great in a conference talk. In practice, they require:

- Every consumer team to actually write and maintain contracts (they won't, at first).
- A culture where producer teams accept that they can't break consumers (some will resist).
- Tooling in CI/CD to enforce contracts (this is the easy part, surprisingly).
- A governance process for resolving conflicts when a producer needs to make a breaking change that violates a contract (this is the hard part, unsurprisingly).

The payoff is worth it. Consumer-driven contracts transform schema evolution from "we hope nothing breaks" to "we know nothing breaks, because the pipeline told us."

---

## The Schema Graveyard: What Happens When Nobody Cleans Up

Let me paint you a picture. It's three years into your event-driven architecture. Your schema registry has 847 schemas across 312 subjects. Your `OrderCreated` event is on version 23. Versions 1 through 17 haven't been produced in over a year, but they're still registered because nobody's sure if there are old events in the topic that use them. There are 14 schemas that contain the word "test" or "temp" in their name. Three schemas are registered under subjects that correspond to topics that no longer exist. Nobody has the full picture. Nobody wants to touch it.

This is the schema graveyard, and it's what happens when you have a schema registry but no schema governance.

### How to Avoid It

1. **Ownership.** Every schema has an owner (a team, not a person). The owner is responsible for its lifecycle, including deprecation and removal.

2. **Deprecation process.** Before removing a schema version, mark it as deprecated. Give consumers a timeline. Check topic retention periods — if the topic has 7-day retention, you only need to keep schemas that were used within the last 7 days.

3. **Automated cleanup.** Write a job that:
   - Identifies schema versions not referenced by any event in the topic (scan the topic headers or schema IDs).
   - Cross-references with consumer group offsets to ensure no consumer will encounter old events.
   - Reports orphaned schemas for review.

4. **Schema lifecycle metadata.** Tag schemas with creation date, owning team, deprecation status, and "last seen in production" timestamps.

```python
# Schema lifecycle tracking
schema_metadata = {
    "subject": "orders-value",
    "version": 17,
    "created_at": "2023-06-15T10:00:00Z",
    "created_by": "order-service-team",
    "status": "deprecated",
    "deprecated_at": "2024-09-01T00:00:00Z",
    "deprecation_reason": "Replaced by v18 — added structured address",
    "removal_eligible_after": "2024-10-01T00:00:00Z",
    "last_seen_in_production": "2024-08-28T14:22:00Z"
}
```

5. **Registry-level metrics.** Track the number of active schemas, the rate of new registrations, the average number of versions per subject, and the number of subjects with no recent activity. Alert when these metrics suggest entropy is winning.

### The Hard Truth

You will not maintain perfect schema hygiene. The graveyard will form. The goal is not prevention but mitigation — keeping the graveyard small, well-mapped, and distinguishable from the schemas that are actually alive.

---

## Code Examples: Compatible vs. Breaking Changes

Let's make this concrete with Avro examples showing what the schema registry will accept and what it will reject.

### Compatible: Adding an Optional Field

```json
// v1
{
  "type": "record",
  "name": "UserRegistered",
  "namespace": "com.example.users",
  "fields": [
    { "name": "userId", "type": "string" },
    { "name": "email", "type": "string" },
    { "name": "registeredAt", "type": "long" }
  ]
}

// v2 — COMPATIBLE under BACKWARD, FORWARD, and FULL
{
  "type": "record",
  "name": "UserRegistered",
  "namespace": "com.example.users",
  "fields": [
    { "name": "userId", "type": "string" },
    { "name": "email", "type": "string" },
    { "name": "registeredAt", "type": "long" },
    { "name": "displayName", "type": ["null", "string"], "default": null }
  ]
}
```

A v1 reader processing a v2 event: ignores `displayName`. A v2 reader processing a v1 event: sets `displayName` to `null`. Everyone's happy.

### Breaking: Adding a Required Field Without a Default

```json
// v1
{
  "type": "record",
  "name": "UserRegistered",
  "fields": [
    { "name": "userId", "type": "string" },
    { "name": "email", "type": "string" }
  ]
}

// v2 — BREAKS BACKWARD COMPATIBILITY
{
  "type": "record",
  "name": "UserRegistered",
  "fields": [
    { "name": "userId", "type": "string" },
    { "name": "email", "type": "string" },
    { "name": "phoneNumber", "type": "string" }
  ]
}
```

A v2 reader processing a v1 event: crashes. Where's `phoneNumber`? There's no default. The Avro deserializer throws. Your registry should reject this if backward compatibility is enforced.

### Breaking: Changing a Field Type

```json
// v1: totalAmount is a double
{ "name": "totalAmount", "type": "double" }

// v2: totalAmount is now a record — BREAKS EVERYTHING
{
  "name": "totalAmount",
  "type": {
    "type": "record",
    "name": "Money",
    "fields": [
      { "name": "amount", "type": "long" },
      { "name": "currency", "type": "string" }
    ]
  }
}
```

This is the kind of change that requires the expand-and-contract pattern. Add `totalAmountV2` as a new field with the structured type, migrate consumers, then deprecate `totalAmount`.

### Breaking: Removing a Field Without a Default (Breaks Forward Compatibility)

```json
// v1
{
  "type": "record",
  "name": "OrderCreated",
  "fields": [
    { "name": "orderId", "type": "string" },
    { "name": "priority", "type": "string" },
    { "name": "totalAmount", "type": "double" }
  ]
}

// v2 — removed priority, BREAKS FORWARD COMPATIBILITY
{
  "type": "record",
  "name": "OrderCreated",
  "fields": [
    { "name": "orderId", "type": "string" },
    { "name": "totalAmount", "type": "double" }
  ]
}
```

A v1 reader (old consumer) processing a v2 event: where's `priority`? If the v1 schema has no default for `priority`, deserialization fails. The fix: `priority` should have had a default value from the start, or you need forward-compatible consumers that tolerate missing fields.

### Protobuf: Safe Field Removal with Reserved Tags

```protobuf
// v1
message OrderCreated {
  string order_id = 1;
  string priority = 2;
  double total_amount = 3;
}

// v2 — removed priority, reserved the tag
message OrderCreated {
  reserved 2;
  reserved "priority";

  string order_id = 1;
  double total_amount = 3;
}
```

In Protobuf, this is safe. Old readers encountering a v2 message will see `priority` as empty string (the default). New readers encountering a v1 message will just skip tag 2. The `reserved` keyword prevents anyone from accidentally reusing tag 2 for a different field in the future, which would be a subtle and devastating bug.

---

## Summary

Schema evolution is not a one-time design decision; it's an ongoing discipline. The choices you make early — serialization format, compatibility level, registry adoption — determine how painful or painless changes will be for the lifetime of the system.

The advice, in brief:

1. **Pick a schema format with built-in evolution support.** Avro or Protobuf. Not raw JSON. Not "we'll be careful."
2. **Use a schema registry.** Enforce compatibility checking in CI/CD and at registration time.
3. **Default to FULL_TRANSITIVE compatibility.** It's restrictive, and that's the point.
4. **Make additive-only changes the norm.** New optional fields with sensible defaults. Boring is beautiful.
5. **Use expand-and-contract for breaking changes.** It takes longer. It's worth it.
6. **Invest in consumer-driven contracts.** They're organizational work disguised as technical work, and they're worth it.
7. **Plan for the graveyard.** Because it's coming whether you plan or not.

Schema evolution is the tax you pay for the privilege of independent deployability. Pay it cheerfully, automate it aggressively, and never, ever skip the compatibility check.
