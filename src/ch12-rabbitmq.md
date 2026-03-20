# RabbitMQ

If Kafka is the event streaming platform that conquered the world through sheer throughput and ambition, RabbitMQ is the message broker that quietly kept the world running while Kafka was getting all the conference keynotes. It is older, more traditional in its design, and refreshingly honest about what it is: a message broker. Not an event streaming platform. Not a distributed commit log. A broker. It accepts messages, routes them according to rules you define, and delivers them to consumers. It does this reliably, flexibly, and with a routing model that remains unmatched in the industry.

RabbitMQ is also a project in an interesting phase of its life — mature, widely deployed, and navigating the transition from its traditional queue-based model toward event streaming capabilities with the introduction of Streams. Whether that transition succeeds in keeping RabbitMQ relevant against Kafka and its competitors is one of the more interesting questions in the messaging space.

---

## Overview

### What It Is

RabbitMQ is an open-source message broker that implements the Advanced Message Queuing Protocol (AMQP). It provides a flexible routing model based on exchanges and bindings, supports multiple messaging protocols, and offers strong delivery guarantees through publisher confirms and consumer acknowledgments.

### Brief History

RabbitMQ was created in 2007 by Rabbit Technologies, a small company founded by Alexis Richardson and Matthias Radestock. The founding premise was to build a proper implementation of the AMQP specification — a protocol designed by JP Morgan Chase and a consortium of financial institutions who were tired of expensive proprietary messaging middleware (IBM MQ, TIBCO).

The implementation language choice was Erlang/OTP, which was unusual then and remains unusual now. Erlang was designed by Ericsson for telecommunications switching systems — highly concurrent, fault-tolerant, soft real-time systems. For a message broker, this is an almost suspiciously good fit. Erlang's lightweight process model, pattern matching, and "let it crash" supervision philosophy translate directly into broker capabilities: millions of concurrent connections, isolated failure handling, and hot code upgrades.

Rabbit Technologies was acquired by VMware in 2010. VMware spun it into Pivotal in 2013. VMware re-acquired Pivotal in 2019. VMware was acquired by Broadcom in 2023. If you are keeping score, that is four corporate parents in fifteen years, which is enough to make any open-source project nervous. Broadcom's acquisition, in particular, raised concerns — Broadcom has a reputation for aggressive cost optimisation of acquired software businesses, and the RabbitMQ community watched carefully for signs of reduced investment. As of this writing, development continues, but the stewardship question is a legitimate factor in long-term planning.

### Who Runs It

RabbitMQ is open source under the Mozilla Public License 2.0. The core team is employed by Broadcom (via the VMware Tanzu division). There is an active community of contributors, but the core development is heavily concentrated in the Broadcom-employed team.

---

## Architecture

### The AMQP Model: Exchanges, Queues, and Bindings

RabbitMQ's routing model is its defining feature and the reason it excels at use cases where Kafka struggles. The model has three components:

**Producers** publish messages to **exchanges**. An exchange is not a queue — it does not store messages. It is a routing engine that examines each incoming message and decides which queues should receive a copy based on **bindings** — rules that connect exchanges to queues.

**Queues** store messages until consumers retrieve them. Unlike Kafka's log, a traditional RabbitMQ queue removes messages once they are acknowledged by a consumer. The message lifecycle is: produced → routed → queued → delivered → acknowledged → gone.

**Bindings** define the routing rules between exchanges and queues. The binding key and the exchange type together determine how messages are routed.

### Exchange Types

This is where RabbitMQ shines:

**Direct exchange**: Routes messages to queues whose binding key exactly matches the message's routing key. Simple, predictable. Use it when you know exactly where a message should go. Think of it as a precise address.

**Topic exchange**: Routes messages based on wildcard pattern matching on the routing key. Routing keys are dot-delimited strings (e.g., `order.placed.us-east`), and binding patterns can use `*` (match one word) and `#` (match zero or more words). So a binding of `order.*.us-east` matches `order.placed.us-east` but not `order.placed.eu-west`, while `order.#` matches everything starting with `order.`. This is extremely powerful for building flexible event routing topologies.

**Fanout exchange**: Routes messages to all bound queues, ignoring the routing key entirely. Every queue gets a copy. This is your pub/sub broadcast mechanism.

**Headers exchange**: Routes based on message header attributes rather than the routing key. Less commonly used but valuable when routing decisions depend on multiple attributes (e.g., "route to this queue if content-type is application/json AND priority is high").

**Default exchange**: A special direct exchange where every queue is automatically bound with a binding key equal to the queue name. Publishing to the default exchange with routing key "my-queue" delivers directly to the queue named "my-queue". This makes RabbitMQ feel like a simple point-to-point queue system when you want it to.

This routing model means you can implement complex event distribution topologies — fan-out to multiple consumers, content-based routing, topic hierarchies — without writing any application-level routing code. The broker handles it. In Kafka, all of this is your problem.

### Queue Types

RabbitMQ has evolved from a single queue implementation to three distinct types, each with different trade-offs.

#### Classic Queues

The original queue type. Messages are stored in memory (with overflow to disk) on a single node. In a cluster, classic queues can be **mirrored** (replicated) to other nodes for high availability, but classic mirrored queues are deprecated as of RabbitMQ 3.13 and should not be used for new deployments.

Classic mirrored queues had several well-documented problems: synchronisation during initial mirroring blocked the queue, adding mirrors to a loaded queue caused backlogs, and the promotion logic during node failures had edge cases that could lose messages. They worked, mostly, but they were a source of operational anxiety.

#### Quorum Queues

The modern replacement for mirrored queues, introduced in RabbitMQ 3.8. Quorum queues use the Raft consensus protocol for replication. They require a majority (quorum) of nodes to be available for writes and guarantee data safety through replicated, durable logs.

Quorum queues are the recommended queue type for any workload that requires high availability and data safety. They are more predictable than mirrored queues, handle node failures more gracefully, and have better performance characteristics under normal operation.

The trade-off: quorum queues use more disk I/O (every message is written to a write-ahead log on all replicas) and do not support some features of classic queues (message TTL per message, queue length limits via drop-head, priorities). For most workloads, these limitations are acceptable.

#### Streams

Introduced in RabbitMQ 3.9, streams are RabbitMQ's answer to the "but can it do what Kafka does?" question. A stream is an append-only log — messages are not removed when consumed. Consumers can read from any point in the stream, replay from the beginning, or start from a timestamp. Sound familiar?

Streams give RabbitMQ event streaming capabilities without requiring a separate system. They support high fan-out (many consumers reading the same data), time-based retention, and offset tracking. The implementation is optimised for sequential disk I/O, borrowing ideas from — you guessed it — Kafka's log design.

We will cover streams in more detail below. The short version: they work, they are improving rapidly, and they make RabbitMQ viable for use cases that previously required Kafka. But they are younger and less battle-tested than Kafka's log.

### The AMQP Protocol

RabbitMQ's native protocol is AMQP 0-9-1, the "original" AMQP that the broker was built to implement. Despite the version number suggesting it is a pre-release specification, AMQP 0-9-1 is a mature, well-defined protocol with broad client support.

RabbitMQ also supports AMQP 1.0 (the OASIS standard, which is a substantially different protocol despite sharing a name), MQTT (for IoT workloads), and STOMP (for text-based simplicity). The multi-protocol support is a genuine differentiator — you can have IoT devices publishing via MQTT and backend services consuming via AMQP from the same broker.

As of RabbitMQ 4.0 (late 2024), AMQP 1.0 has become a first-class citizen alongside 0-9-1, with native support for streams and quorum queues over the 1.0 protocol. This is significant because AMQP 1.0 is the protocol that cloud providers and enterprise middleware vendors have standardised on.

### Acknowledgments and Publisher Confirms

#### Consumer Acknowledgments

When a consumer receives a message, it must **acknowledge** (ack) it to tell the broker the message was successfully processed. Until the ack is received, the message stays in the queue and will be redelivered if the consumer disconnects. This is the foundation of at-least-once delivery.

Consumers can also **reject** (nack) a message, optionally requesting requeue (put it back in the queue for another attempt) or dead-lettering (route it to a designated dead-letter exchange for error handling).

Manual acknowledgment with `basic.ack` after successful processing is the safe default. Auto-acknowledgment (the broker considers the message delivered as soon as it sends it) is at-most-once delivery and appropriate only for non-critical messages.

#### Publisher Confirms

The producer-side equivalent of consumer acks. When publisher confirms are enabled on a channel, the broker sends a confirmation (or negative confirmation) to the producer after the message has been durably stored. This closes the "I published a message but don't know if the broker actually received it" gap.

Without publisher confirms, a message could be lost between the producer sending it and the broker persisting it — network failure, broker crash, or just a full queue with a reject policy. Publisher confirms are essential for any workflow where message loss is unacceptable.

### Clustering

RabbitMQ clustering connects multiple nodes into a single logical broker. Cluster metadata (exchange definitions, queue definitions, bindings, users, policies) is replicated to all nodes. Queue data (the actual messages) is *not* automatically replicated — you need quorum queues or streams for data replication.

Clustering gives you:
- **Horizontal scaling**: Distribute queues across nodes to spread load.
- **High availability**: With quorum queues, survive node failures without message loss.
- **Unified management**: A single management interface for the entire cluster.

Clustering requires reliable, low-latency networking. RabbitMQ clusters should be deployed within a single datacenter or availability zone. For cross-datacenter replication, use federation or shovel.

### Federation and Shovel

**Federation** links exchanges or queues across RabbitMQ clusters (or individual nodes) that may be geographically distributed. Federated exchanges forward messages to downstream exchanges based on bindings. Federated queues allow consumers on one cluster to consume from a queue on another.

Federation is asynchronous, tolerates WAN latency and intermittent connectivity, and does not require the clusters to share the same Erlang cookie (authentication secret). It is designed for cross-datacenter and cross-region scenarios.

**Shovel** is a simpler mechanism: it is a built-in plugin that acts as a consumer on one broker and a producer on another, forwarding messages between them. Less intelligent than federation, but simpler and more flexible — you can shovel between any two AMQP endpoints, including non-RabbitMQ brokers.

---

## Strengths

### Routing Flexibility

No other mainstream broker matches RabbitMQ's routing model. Topic exchanges with wildcard bindings, headers-based routing, and exchange-to-exchange bindings give you content-based message routing that would require custom application code in any other system. If your use case involves directing messages to different consumers based on message attributes, RabbitMQ is the natural choice.

### Mature Protocol Support

AMQP 0-9-1 is a well-understood, widely implemented protocol. The multi-protocol support (AMQP 1.0, MQTT, STOMP) means RabbitMQ can serve as a polyglot messaging layer for heterogeneous environments. This is particularly valuable in enterprise settings where different systems speak different protocols.

### Ease of Getting Started

RabbitMQ has one of the best out-of-box experiences of any message broker. Install it, start it, open the management UI, and you have a working broker with a web-based dashboard for creating exchanges, queues, bindings, publishing test messages, and monitoring. The learning curve from "nothing installed" to "processing messages" is measured in minutes, not hours.

### Management UI and Observability

The built-in management UI is genuinely useful — not just a toy dashboard, but a tool that operators use daily. It shows queue depths, message rates, connection counts, consumer utilisation, and node health. It also exposes an HTTP API for programmatic management.

Prometheus metrics are available via a built-in plugin, with well-maintained Grafana dashboards. The observability story is solid.

### Plugin Ecosystem

RabbitMQ has a mature plugin system. Notable plugins include the management UI, Prometheus metrics, federation, shovel, MQTT support, STOMP support, tracing, and delayed message exchange. The plugin architecture means you can extend the broker's functionality without forking it.

### Quorum Queues

The introduction of quorum queues was a turning point for RabbitMQ's reliability story. Raft-based replication provides predictable, well-understood behaviour during node failures. If you are deploying RabbitMQ today, quorum queues should be your default for any queue that matters.

---

## Weaknesses

### Throughput Ceiling

RabbitMQ is not designed for the same throughput as Kafka. A well-tuned RabbitMQ cluster can handle tens of thousands of messages per second per queue — respectable, but an order of magnitude less than what Kafka achieves. The routing layer, per-message acknowledgment overhead, and queue-based storage model all contribute to this ceiling.

For many workloads, tens of thousands of messages per second is plenty. But if you need hundreds of thousands or millions, RabbitMQ is not the right tool, and no amount of tuning will change that.

### Queue Depth Problems

When consumers fall behind and queues grow deep, RabbitMQ suffers. Large queues increase memory usage, slow down message delivery (because the broker is managing more state), and can trigger memory alarms that block publishers. The broker is designed for queues that are relatively short — messages flow in and out quickly. Long queues with millions of messages are a sign that something is wrong.

This is a fundamental design difference from Kafka, where retaining millions of messages is the normal operating mode. RabbitMQ's storage model is optimised for message throughput, not message retention.

### Erlang Operational Expertise

Erlang is a fantastic language for building RabbitMQ. It is a less fantastic language for debugging RabbitMQ. When things go wrong at the system level — processes accumulating, memory growing, nodes failing to cluster — understanding what is happening requires familiarity with Erlang's process model, OTP supervision trees, and the Erlang VM's (BEAM) behaviour under stress.

You do not need to *write* Erlang to operate RabbitMQ. But when you need to read Erlang crash logs, interpret process dump output, or understand why the Erlang distribution protocol is rejecting connections, you will wish you had someone on the team who speaks the language.

### No Native Message Replay

Traditional RabbitMQ queues delete messages after acknowledgment. If you need to reprocess historical messages — because you found a bug, deployed a new consumer, or want to rebuild state — those messages are gone. You either need a separate archival system, or you use RabbitMQ Streams (which do support replay, but are a different thing from queues).

This is a significant limitation for event-driven architectures that rely on replay capability. It is also the primary reason teams choose Kafka over RabbitMQ for event sourcing and stream processing workloads.

### Classic Mirrored Queue Legacy

If you are running an older RabbitMQ deployment with classic mirrored queues, you are carrying technical debt. Mirrored queues are deprecated and will be removed in a future release. Migration to quorum queues is well-supported but requires planning — the queue types have different semantics, and some applications may depend on features that quorum queues do not support.

---

## Ideal Use Cases

- **Task queues and work distribution**: Distribute tasks to a pool of workers with acknowledgment-based reliability. This is RabbitMQ's bread and butter.
- **Complex routing topologies**: Route messages based on content, headers, or topic patterns without custom code.
- **Request-reply patterns**: RabbitMQ has first-class support for RPC-style messaging with reply-to queues and correlation IDs.
- **Multi-protocol environments**: IoT devices on MQTT, backend services on AMQP, legacy systems on STOMP — all on one broker.
- **Microservice command bus**: Distributing commands (not events) to specific services, with routing and acknowledgment.
- **Moderate-throughput event distribution**: When you need pub/sub but do not need Kafka-scale throughput or log-based retention.

---

## Operational Reality

### Memory and Disk Alarms

RabbitMQ has a built-in flow control mechanism tied to resource limits:

- **Memory alarm**: When the broker's memory usage exceeds the configured threshold (default 40% of system RAM), all publishers are blocked. Consumers continue to drain queues, but no new messages are accepted until memory drops below the threshold. This is aggressive but effective — it prevents the broker from running out of memory and crashing.
- **Disk alarm**: When free disk space drops below the configured threshold (default 50MB, which is far too low for production — set it higher), the same publisher blocking occurs.

These alarms are your early warning system. If they fire frequently, your queues are too deep, your consumers are too slow, or your cluster is undersized.

### Queue Depth Monitoring

The single most important metric in RabbitMQ operations is queue depth per queue. A queue that is growing means consumers are not keeping up. A queue with millions of messages means you have a problem that is getting worse. Unlike Kafka, where large backlogs are normal and expected, a deep RabbitMQ queue is a symptom that needs attention.

### Upgrade Strategies

RabbitMQ supports rolling upgrades within certain version ranges. The process is:

1. Stop one node.
2. Upgrade the binary.
3. Start the node and let it rejoin the cluster.
4. Wait for quorum queues to synchronise.
5. Repeat for each node.

Major version upgrades (e.g., 3.x to 4.x) may require feature flags to be enabled sequentially and can involve more significant changes to configuration and plugin compatibility. Read the release notes. All of them.

The Erlang version also matters — RabbitMQ has specific Erlang version requirements, and upgrading RabbitMQ sometimes requires upgrading Erlang first.

### Cluster Sizing

A typical production RabbitMQ cluster is 3 nodes with quorum queues (replication factor 3). This gives you majority-based fault tolerance — you can lose one node without losing data or availability.

For higher throughput, add nodes and distribute queues across them. Unlike Kafka, where partitions provide automatic parallelism within a topic, RabbitMQ requires you to manage queue distribution yourself (or use consistent hash exchange plugins for automatic sharding).

Memory sizing depends heavily on queue depth and message size. A node processing messages quickly (short queues) needs less memory than one with deep backlogs. Start with 4-8GB per node for moderate workloads and monitor from there.

### Managed Offerings

- **CloudAMQP**: The most established RabbitMQ-as-a-service provider. Multi-cloud, solid management interface, good support.
- **Amazon MQ for RabbitMQ**: AWS-managed, limited configuration options, simpler but less flexible.
- **VMware Tanzu RabbitMQ**: Commercial distribution with additional features for enterprise environments.
- **Azure Service Bus**: Not RabbitMQ, but supports AMQP and is often considered as an alternative in Azure environments.

---

## RabbitMQ Streams in Depth

Streams deserve special attention because they represent RabbitMQ's strategic response to the event streaming trend.

A stream is an append-only, immutable log — the same abstraction as a Kafka topic partition. Messages are written once and retained based on time or size limits. Consumers can attach at any point in the stream, read forward, and track their offset.

Streams use a custom binary protocol (separate from AMQP) optimised for high throughput and low overhead. They leverage sequential disk I/O, memory-mapped files, and a purpose-built storage engine.

### When to Use Streams vs Queues

- **Streams**: Large fan-out (many consumers reading the same data), replay requirements, high-throughput write-once-read-many workloads.
- **Queues**: Task distribution (each message processed by one consumer), complex routing via exchanges, message-level TTL and priority.

Streams are not a replacement for queues. They are a complementary tool for a different set of use cases. The power of modern RabbitMQ is that you can use both in the same cluster, with the same management tools, and even route between them using exchanges.

### Limitations

Streams are younger than Kafka's log implementation and have fewer features. Sub-second filtering within streams is limited. The ecosystem around streams (connectors, stream processing) is not comparable to Kafka's. They are improving with each release, but as of now, if event streaming is your primary use case, Kafka remains the more complete solution.

---

## Code Examples

### Python Producer (pika)

```python
import pika
import json

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost')
)
channel = connection.channel()

# Declare exchange and queue
channel.exchange_declare(exchange='order-events', exchange_type='topic', durable=True)
channel.queue_declare(queue='order-processing', durable=True)
channel.queue_bind(
    queue='order-processing',
    exchange='order-events',
    routing_key='order.placed.*'  # Match all regions
)

# Enable publisher confirms for durability
channel.confirm_delivery()

message = json.dumps({
    'type': 'OrderPlaced',
    'orderId': 'order-7829',
    'region': 'us-east',
})

try:
    channel.basic_publish(
        exchange='order-events',
        routing_key='order.placed.us-east',
        body=message,
        properties=pika.BasicProperties(
            delivery_mode=2,  # Persistent message
            content_type='application/json',
        ),
    )
    print('Message published and confirmed')
except pika.exceptions.UnroutableError:
    print('Message could not be routed')

connection.close()
```

### Python Consumer (pika)

```python
import pika
import json

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost')
)
channel = connection.channel()

channel.queue_declare(queue='order-processing', durable=True)

# Fair dispatch — don't send more than one message to a worker at a time
channel.basic_qos(prefetch_count=1)

def on_message(channel, method, properties, body):
    event = json.loads(body)
    print(f"Processing order: {event['orderId']}")

    # Process the message...

    # Acknowledge after successful processing
    channel.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(
    queue='order-processing',
    on_message_callback=on_message,
    auto_ack=False,  # Manual acknowledgment
)

print('Waiting for messages...')
channel.start_consuming()
```

### Java Producer (Spring AMQP)

```java
import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitConfig {

    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange("order-events");
    }

    @Bean
    public Queue orderProcessingQueue() {
        return QueueBuilder.durable("order-processing")
            .quorum()  // Use quorum queue for replication
            .build();
    }

    @Bean
    public Binding orderBinding(Queue orderProcessingQueue,
                                 TopicExchange orderExchange) {
        return BindingBuilder
            .bind(orderProcessingQueue)
            .to(orderExchange)
            .with("order.placed.*");
    }
}

// In your service:
@Service
public class OrderEventPublisher {

    private final RabbitTemplate rabbitTemplate;

    public OrderEventPublisher(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    public void publishOrderPlaced(String orderId, String region) {
        String routingKey = "order.placed." + region;
        String message = String.format(
            "{\"type\":\"OrderPlaced\",\"orderId\":\"%s\"}", orderId
        );

        rabbitTemplate.convertAndSend("order-events", routingKey, message);
    }
}
```

### Java Consumer (Spring AMQP)

```java
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component
public class OrderEventListener {

    @RabbitListener(queues = "order-processing")
    public void handleOrderPlaced(String message) {
        System.out.println("Received: " + message);

        // Process the order event...

        // Acknowledgment is automatic with Spring's default settings
        // (acknowledged after the method returns without exception)
    }
}
```

---

## Verdict

RabbitMQ is the right choice when your problem is *routing messages* rather than *streaming events*. Its exchange-binding-queue model is the most expressive routing system in the messaging world, and for task distribution, RPC patterns, and complex event routing, nothing matches it.

**Pick RabbitMQ when:**
- You need flexible message routing based on content, topics, or headers
- Your primary pattern is task distribution (work queues with acknowledgment)
- You need request-reply / RPC messaging
- You want a broker that is easy to set up, manage, and understand
- You need multi-protocol support (AMQP, MQTT, STOMP)
- Your throughput requirements are moderate (tens of thousands of messages/sec)
- You need a broker that your team can operate without a PhD in distributed systems

**Avoid RabbitMQ when:**
- You need high-throughput event streaming (hundreds of thousands+ messages/sec)
- You need log-based message retention and replay as a core feature (though Streams are closing this gap)
- You need built-in stream processing (Kafka Streams, ksqlDB)
- Your primary workload is event sourcing with long-term retention
- You need a massive connector ecosystem for data integration

RabbitMQ is not trying to be Kafka, and that is its greatest strength. It is a message broker — arguably the best general-purpose message broker available — and for the workloads it was designed for, it remains an excellent choice. The addition of Streams extends its relevance into event streaming territory, but that feature is still maturing. If your workload is primarily event streaming, evaluate RabbitMQ Streams honestly against your requirements rather than assuming the name alone is sufficient.
