# Memphis

Every few years, a new messaging project arrives with a fresh perspective on developer experience. The pitch usually goes something like: "Message brokers are too complicated. What if we made one that developers actually enjoy using?" Memphis is (or was) one of these projects, and it is worth examining both for what it tried to do and for the cautionary tale it represents about the lifecycle of newer open-source infrastructure projects.

Let us be direct from the start: Memphis has had a turbulent trajectory. The company behind it (Memphis.dev) pivoted, the open-source project's future became uncertain, and as of the time of writing, its long-term viability is a genuine question. We will cover the technology honestly — it had real ideas worth understanding — but we will also be honest about the risks. If you are evaluating Memphis for a new project, read the entire chapter, especially the end.

---

## Overview

Memphis was created by Memphis.dev (originally based in Israel) with the stated goal of making message brokers accessible to developers who were not distributed systems specialists. The founding insight was that Kafka, RabbitMQ, and Pulsar all required significant operational knowledge and architectural understanding before they could be used productively. Memphis aimed to lower that barrier.

The project launched around 2021-2022 and gained initial traction through a developer-friendly GUI, built-in schema management, dead-letter handling, and a focus on making common tasks — producing messages, consuming them, handling failures — simple by default rather than simple after reading three books and attending a conference talk.

Memphis was built on top of NATS JetStream, which is a significant architectural decision. Rather than building a storage engine, replication protocol, and networking layer from scratch, Memphis used NATS JetStream as its foundation and added a developer experience layer on top. This gave it the performance and reliability characteristics of NATS while allowing the Memphis team to focus on UX, tooling, and higher-level features.

### The Pivot and the Present

Memphis.dev as a company underwent significant changes. The open-source project's development slowed. The company pivoted toward different products. The GitHub repository's commit frequency dropped. Community activity declined.

This is not unusual in the world of venture-backed open-source infrastructure. Companies need revenue, and developer experience layers over existing technology are difficult to monetise when the underlying technology (NATS JetStream) is freely available and well-documented. The business model challenge — "we make NATS easier to use, please pay us" — proved difficult to sustain.

As of this writing, Memphis should be evaluated with caution. The technology works, the ideas are sound, but the project's future maintenance and development are uncertain. If you are building production systems with a multi-year horizon, you need confidence that your messaging infrastructure will be maintained, patched, and improved. That confidence is easier to have with NATS, Kafka, or RabbitMQ than with Memphis.

We cover it here because the ideas deserve examination, and because understanding why projects like Memphis emerge (and struggle) tells you something important about the messaging landscape.

---

## Architecture

### Built on NATS JetStream

Memphis is, architecturally, a layer on top of NATS JetStream. If you have read Chapter 17 on NATS, you already understand the foundation:

- **NATS** provides the core publish-subscribe messaging, connection management, and clustering.
- **JetStream** adds persistence, exactly-once delivery, consumer groups, and stream management.
- **Memphis** adds a developer experience layer: a GUI, schema management, dead-letter handling, SDK abstractions, and operational tooling.

The Memphis broker is a modified NATS server with additional components:
- A metadata store (using JetStream internally)
- A REST API for management operations
- A web-based GUI
- Schema enforcement logic
- Dead-letter station management
- SDK-facing protocol handling

Messages produced to Memphis flow through to JetStream streams. Consumers reading from Memphis are reading from JetStream consumers. The Memphis layer intercepts, decorates, and manages the lifecycle, but the heavy lifting — storage, replication, delivery guarantees — is JetStream.

### Stations

Memphis uses the concept of a "station" rather than "topic" or "queue." A station is the fundamental unit of message organization. Under the hood, a station maps to a JetStream stream with specific configuration.

Stations have:
- **Retention policy**: time-based, size-based, or message-count-based
- **Replication factor**: how many nodes store copies (inherits from JetStream)
- **Schema enforcement**: optional schema validation on produce
- **Dead-letter station**: where failed messages go after retry exhaustion

The station abstraction is Memphis's primary organisational concept, and it is intentionally simpler than the topic/partition/consumer-group hierarchy of Kafka or the exchange/binding/queue model of RabbitMQ. For developers coming to messaging for the first time, this simplicity is a genuine benefit.

### Dead-Letter Station

One of Memphis's more thoughtful features. When a consumer fails to process a message after a configurable number of retries, the message is moved to a dead-letter station (DLS). The DLS is visible in the GUI, where operators can:

- Inspect failed messages
- See the failure reason (if the consumer SDK reports it)
- Resend messages to the original station
- Drop messages

This is functionality that you *can* build with Kafka or RabbitMQ, but Memphis makes it a first-class feature with built-in tooling. For teams that have experienced the joy of discovering that a consumer has been silently failing for three days because nobody checked the dead-letter queue, this is appreciated.

### Poison Message Detection

Related to dead-letter handling, Memphis tracks messages that repeatedly cause consumer failures. These "poison messages" — messages that crash consumers, trigger exceptions, or time out repeatedly — are flagged in the UI and can be automatically routed to the DLS.

In a typical Kafka setup, a poison message can be insidious: it arrives, the consumer crashes, the consumer restarts, it reads the same message (because the offset was not committed), crashes again, and enters a restart loop. Memphis's poison message detection breaks this cycle by identifying the problematic message and removing it from the normal processing path.

---

## Key Features

### Schema Management (Schemaverse)

Memphis included built-in schema management called "Schemaverse." Schemas could be attached to stations, and the broker would validate messages against the schema on produce. Supported formats included:

- Protobuf
- JSON Schema
- GraphQL (an unusual choice that reflected Memphis's developer-experience focus)
- Avro

Schema enforcement at the broker level — rather than relying on client-side validation or a separate Schema Registry — simplifies the architecture. You do not need a separate Confluent Schema Registry deployment; schema management is part of the broker.

The trade-off is coupling: your schema management is tied to your broker. With Kafka + Confluent Schema Registry, you can swap brokers without losing your schema management. With Memphis, the schema management goes where Memphis goes.

### GUI

The Memphis GUI is perhaps the feature that best embodied the project's philosophy. It provided:

- Real-time station monitoring (message rates, consumer lag)
- Message browsing and inspection
- Schema management
- Dead-letter station management
- User and permission management
- System health overview

For a developer who has just set up their first message broker and wants to understand what is happening, a well-designed GUI is enormously valuable. The Kafka ecosystem has various UIs (Kafka UI, AKHQ, Confluent Control Center), but none are as tightly integrated with the broker as Memphis's GUI was with Memphis.

### SDK Design

Memphis SDKs were designed for simplicity. A minimal producer/consumer in Memphis required fewer lines of code and fewer concepts than the equivalent in Kafka or RabbitMQ. The SDKs handled connection management, reconnection, and basic error handling internally.

This is a double-edged sword. Simple SDKs are great for getting started and terrible for debugging production issues. When the SDK handles reconnection transparently, you do not know it is happening. When it manages consumer acknowledgment internally, you may not realise that your processing guarantees depend on SDK configuration you never examined.

---

## Strengths

**Developer Experience.** This was Memphis's genuine differentiator. The combination of a clean GUI, simple SDKs, built-in dead-letter handling, and schema management created a "batteries included" experience that lowered the barrier to entry for event-driven architecture. For teams where nobody has operated a message broker before, this matters.

**Built-in Observability.** Station metrics, consumer lag, poison message detection, and dead-letter monitoring were all available out of the box. No Prometheus exporter to configure, no Grafana dashboards to import. You could see what was happening in your messaging system by opening a browser.

**Low Learning Curve.** Memphis required understanding one concept (stations) to get started, versus Kafka's topics/partitions/consumer-groups/offsets or RabbitMQ's exchanges/bindings/queues/acknowledgments. For educational purposes and rapid prototyping, this was valuable.

**Poison Message Handling.** The automatic detection and routing of problematic messages is a genuinely useful feature that most messaging systems leave to the user to implement.

**NATS Foundation.** By building on NATS JetStream, Memphis inherited solid performance characteristics, clustering, and persistence without having to build them from scratch. NATS is a proven, well-engineered system.

---

## Weaknesses

**Project Viability.** This is the elephant in the room. A messaging system is foundational infrastructure — it is not something you swap out easily. Adopting a project with uncertain maintenance means accepting the risk that you may need to migrate to something else in the future. Migration is always more expensive than anyone estimates.

**Smaller Community.** Even at its peak, Memphis's community was small relative to Kafka, RabbitMQ, or even NATS. Fewer users means fewer bug reports, fewer contributed fixes, less operational knowledge shared in blog posts and conference talks, and fewer people on your team who have experience with it.

**NATS Dependency.** Memphis's strength (building on NATS) was also a constraint. Memphis was limited by what JetStream provided. Features that required going beyond JetStream's capabilities were difficult to implement. And the question "why not just use NATS JetStream directly?" was always lurking — if you are going to learn the underlying system anyway when things go wrong, why not start there?

**Enterprise Features.** Advanced features — SSO integration, role-based access control, advanced monitoring — were positioned as enterprise/commercial features. For a project in Memphis's situation, this creates a tension: the open-source version needs to be compelling enough to drive adoption, but the commercial version needs to be differentiated enough to generate revenue.

**Scale Ceiling.** Memphis targeted small-to-medium workloads. For high-throughput, large-scale event streaming, Kafka, Redpanda, or even raw NATS JetStream were better choices. Memphis's abstraction layer added overhead that, while negligible at moderate scale, became noticeable at high volumes.

---

## Ideal Use Cases

**Teams New to Event-Driven Architecture.** If your team has never operated a message broker and you want to start exploring event-driven patterns, Memphis (or something like it) lowers the entry barrier. The GUI and simplified concepts help build understanding before graduating to more complex systems.

**Rapid Prototyping.** When you need messaging for a prototype or proof of concept and you want to focus on application logic rather than infrastructure, Memphis's quick setup and simple SDKs are helpful.

**Development and Testing Environments.** Memphis's low resource requirements and easy setup make it suitable for local development and CI/CD testing, even if production uses a different system.

**Internal Tooling.** For internal applications with moderate scale requirements and a premium on developer productivity, Memphis's trade-offs (simplicity over scale) can be the right ones.

---

## Operational Reality

### Deployment

Memphis could be deployed via Docker, Docker Compose, Kubernetes (Helm chart), or as a managed cloud service (Memphis Cloud, when it was operational).

```bash
# Docker Compose deployment (the simplest path)
curl -s https://memphisdev.github.io/memphis-docker/docker-compose.yml \
  -o docker-compose.yml
docker compose up -d
```

```yaml
# Kubernetes via Helm
helm repo add memphis https://k8s.memphis.dev/charts/
helm install memphis memphis/memphis --namespace memphis --create-namespace
```

The Docker Compose deployment spun up the Memphis broker, the GUI (on port 9000 by default), and the required metadata storage. For development, it was genuinely simple. For production, the Kubernetes deployment was the recommended path.

### Monitoring

Monitoring was primarily through the built-in GUI. For external monitoring, Memphis exposed metrics that could be scraped by Prometheus. However, the monitoring story was less mature than Kafka's or NATS's extensive metric ecosystems. You had the GUI (good for humans) and basic Prometheus metrics (adequate for alerting), but deep operational introspection required understanding the underlying NATS JetStream internals.

### What Happens When Things Go Wrong

This is where Memphis's abstraction layer became a liability. When a station was slow, was it a Memphis issue or a JetStream issue? When consumers were lagging, was the bottleneck in Memphis's SDK layer or in the underlying NATS consumer? Debugging required understanding both Memphis's layer *and* NATS JetStream, which undermined the "simplicity" value proposition.

Experienced operators would often bypass Memphis's tooling and use NATS's native monitoring (`nats` CLI, JetStream management API) to diagnose issues. This is the inevitable fate of abstraction layers: they help until they do not, and then you need to understand what is underneath.

---

## Memphis vs NATS JetStream Directly

This is the question that haunted Memphis from the start: when does the abstraction pay for itself?

**Choose Memphis when:**
- Your team has no NATS experience and wants a gentler on-ramp
- The built-in GUI, dead-letter handling, and schema management save you from building these features yourself
- Your workload is moderate and the abstraction overhead is negligible
- Developer experience is a higher priority than operational transparency

**Choose NATS JetStream directly when:**
- Your team can invest time learning NATS (it is not that hard)
- You want full control over configuration and behaviour
- You need maximum performance without abstraction overhead
- You want the confidence of a large, active, well-maintained project
- You are building production systems with a multi-year horizon

The honest assessment: for most teams, learning NATS JetStream directly is the better long-term investment. The learning curve is modest, the documentation is good, and you are investing in knowledge of a system that will be maintained and improved for years to come. Memphis's developer experience advantages — while real — are most valuable in the first few weeks of adoption. After that, you need operational depth, and that depth comes from understanding the underlying system.

---

## Code Examples

### Python

```python
import asyncio
from memphis import Memphis, MemphisError, MemphisConnectError

async def producer_example():
    memphis = Memphis()
    try:
        await memphis.connect(
            host="localhost",
            username="root",
            password="memphis",
            account_id=1
        )

        producer = await memphis.producer(
            station_name="orders",
            producer_name="order-service"
        )

        event = {
            "type": "OrderPlaced",
            "orderId": "ord-7829",
            "amount": 149.99,
            "currency": "USD"
        }

        # Memphis SDK handles serialization
        await producer.produce(
            message=event,
            headers={"event_type": "OrderPlaced"}
        )
        print("Message produced successfully")

    except (MemphisError, MemphisConnectError) as e:
        print(f"Error: {e}")
    finally:
        await memphis.close()


async def consumer_example():
    memphis = Memphis()
    try:
        await memphis.connect(
            host="localhost",
            username="root",
            password="memphis",
            account_id=1
        )

        consumer = await memphis.consumer(
            station_name="orders",
            consumer_name="payment-service",
            consumer_group="payment-group"
        )

        # Callback-based consumption
        async def message_handler(messages, error, context):
            if error:
                print(f"Consumer error: {error}")
                return

            for msg in messages:
                try:
                    data = msg.get_data()
                    print(f"Received: {data}")
                    # Acknowledge successful processing
                    await msg.ack()
                except Exception as e:
                    print(f"Processing error: {e}")
                    # Message will be redelivered (and eventually
                    # sent to DLS if retries are exhausted)

        # Start consuming
        consumer.consume(message_handler)

        # Keep the consumer running
        await asyncio.sleep(60)

    except (MemphisError, MemphisConnectError) as e:
        print(f"Error: {e}")
    finally:
        await memphis.close()


if __name__ == "__main__":
    asyncio.run(producer_example())
    asyncio.run(consumer_example())
```

### Node.js

```javascript
const { Memphis } = require("memphis-dev");

async function producerExample() {
  const memphis = new Memphis();

  try {
    await memphis.connect({
      host: "localhost",
      username: "root",
      password: "memphis",
      accountId: 1,
    });

    const producer = await memphis.producer({
      stationName: "orders",
      producerName: "order-service",
    });

    const event = {
      type: "OrderPlaced",
      orderId: "ord-7829",
      amount: 149.99,
      currency: "USD",
    };

    await producer.produce({
      message: Buffer.from(JSON.stringify(event)),
      headers: { event_type: "OrderPlaced" },
    });

    console.log("Message produced successfully");
  } catch (error) {
    console.error("Producer error:", error);
  } finally {
    await memphis.close();
  }
}

async function consumerExample() {
  const memphis = new Memphis();

  try {
    await memphis.connect({
      host: "localhost",
      username: "root",
      password: "memphis",
      accountId: 1,
    });

    const consumer = await memphis.consumer({
      stationName: "orders",
      consumerName: "payment-service",
      consumerGroup: "payment-group",
    });

    consumer.on("message", (message, context) => {
      const data = JSON.parse(message.getData().toString());
      console.log("Received:", data);

      // Acknowledge the message
      message.ack();
    });

    consumer.on("error", (error) => {
      console.error("Consumer error:", error);
    });

    // The consumer runs until explicitly stopped
    console.log("Consumer listening...");
  } catch (error) {
    console.error("Setup error:", error);
    await memphis.close();
  }
}

// Run
producerExample().then(() => consumerExample());
```

Note the relative simplicity compared to Kafka client code. No partition configuration, no serializer classes, no consumer group rebalancing callbacks. Memphis's SDK handles these details internally. Whether this simplicity is a benefit (less boilerplate) or a liability (less control) depends on your needs and your comfort with implicit behaviour.

---

## The Broader Lesson

Memphis represents a pattern worth understanding: the developer experience layer over infrastructure. Several projects have attempted this in the messaging space — making brokers more accessible, wrapping complexity in friendly UIs, providing opinionated defaults.

These projects face a structural challenge. The underlying infrastructure (NATS, in Memphis's case) is itself improving its developer experience. NATS added better documentation, a management CLI, and monitoring tools. As the foundation improves, the value of the abstraction layer shrinks. And the abstraction layer has a maintenance cost that the underlying project does not bear.

This does not mean developer experience does not matter — it profoundly does. But it suggests that developer experience improvements are more sustainable when they are part of the core project rather than a separate layer on top. The NATS team investing in better documentation and tooling is more durable than a separate company building a wrapper.

---

## Verdict

Memphis had genuinely good ideas. The developer experience focus, the built-in dead-letter handling, the schema management, the GUI — these were real improvements over the status quo of "here is a broker, good luck." The project demonstrated that messaging infrastructure could be more approachable without being less capable.

However, good ideas are necessary but not sufficient. A messaging system is infrastructure that you commit to for years. It needs sustained development, an active community, responsive security patching, and a credible roadmap. Memphis's uncertain trajectory makes it a risky choice for production systems today.

If you are attracted to what Memphis offered, the practical recommendation is:

1. **Use NATS JetStream directly.** You get the same underlying engine, better long-term viability, and a larger community. The learning curve is modest.

2. **Build the missing pieces.** If you want dead-letter handling, build it on top of JetStream (it is a few dozen lines of code). If you want a GUI, use NATS's management tools or third-party dashboards.

3. **Watch the space.** Developer experience in messaging is an unsolved problem. The next project that tackles it might come from within an established broker's ecosystem rather than as a separate layer, and that would be more durable.

Memphis was a worthwhile experiment. It asked the right questions about developer experience in event-driven architecture. The answers it provided were good but may not outlast the project itself. In infrastructure, the best technology does not always win — the best-maintained technology does. Choose accordingly.
