# Apache Kafka

If event-driven architecture has a mascot, it is Apache Kafka. Not because it was first — it was not — but because it achieved something rare in infrastructure software: it became the default. When someone says "we need a message broker," the next sentence is usually "so, Kafka?" regardless of whether Kafka is appropriate for the workload in question. This is a testament to both its genuine capabilities and its formidable marketing apparatus.

Kafka deserves its reputation as a powerful, battle-tested platform for event streaming at scale. It also deserves an honest assessment of its sharp edges, operational demands, and the significant gap between a "Hello World" producer and a production-grade deployment. This chapter provides both.

---

## Overview

### What It Is

Apache Kafka is a distributed event streaming platform. At its core, it is a distributed, partitioned, replicated commit log with pub/sub semantics bolted on top. It was designed to be the central nervous system for data — a unified platform that handles real-time event streams and historical data replay with equal competence.

### Brief History

Kafka was born at LinkedIn in 2010, created by Jay Kreps, Neha Narkhede, and Jun Rao. LinkedIn needed to move massive amounts of data — user activity events, metrics, logs — between systems in real time, and nothing on the market did what they needed at the scale they needed it.

The key insight was deceptively simple: model the message broker as an append-only log. Instead of the traditional message queue model (message arrives, consumer processes it, message disappears), Kafka retains messages for a configurable period. Consumers track their own position (offset) in the log. This means multiple consumers can read the same data independently, consumers can rewind and replay, and the broker does not need to track per-consumer state.

Kreps named it after Franz Kafka, the author, because "it is a system optimised for writing." Whether the existential dread of operating it in production was intentional homage is left as an exercise for the reader.

Kafka was open-sourced under the Apache License in 2011, became an Apache Top-Level Project in 2012, and Kreps, Narkhede, and Rao founded Confluent in 2014 to build a commercial platform around it. Confluent has been enormously successful, going public in 2021, and has become the primary driver of Kafka's development — for better and for worse, as the line between "open source Kafka" and "Confluent Platform" can be blurry.

### Who Runs It

The Apache Software Foundation governs the open-source project. Confluent employs most of the core committers. This creates the usual tension of a corporate-backed open-source project: Confluent has a financial incentive to add premium features to their commercial offering rather than to the open-source core. To date, this has not crippled the community edition, but it is worth being aware of.

---

## Architecture

### The Commit Log

Everything in Kafka flows from a single abstraction: the append-only commit log. A Kafka **topic** is a named log. Producers append records to the end. Consumers read from a position (offset) and move forward. Records are immutable once written. The log is retained for a configurable period (time-based or size-based) and then old segments are deleted or compacted.

This is powerful because:

- **Decoupling in time.** Consumers do not need to be running when messages are produced. They catch up later.
- **Multiple consumers.** Different consumer groups can read the same topic independently at different speeds.
- **Replay.** Reset a consumer's offset to the beginning and reprocess everything. This is invaluable for bug fixes, new consumer deployments, and data reprocessing.
- **Ordering.** Within a single partition, records are strictly ordered by offset.

### Brokers and Clusters

A Kafka cluster consists of multiple **broker** nodes. Each broker is a JVM process that handles read/write requests, manages partitions, and replicates data. Brokers are stateful — they store data on local disks.

A production cluster typically runs at least three brokers for replication. Large deployments run dozens or hundreds.

### Partitions

Each topic is divided into one or more **partitions**. A partition is a single, ordered, append-only log. Partitions are the unit of parallelism in Kafka:

- Producers can write to different partitions concurrently.
- Each partition in a consumer group is consumed by exactly one consumer instance, so more partitions means more consumer parallelism.
- Partitions are distributed across brokers for load balancing.

The partition count is set at topic creation and is *very* difficult to change later. Increasing partitions is possible but reshuffles key-based routing. Decreasing partitions is not supported without recreating the topic. Choose wisely, or more realistically, over-provision slightly and hope for the best.

### Replication and ISR

Each partition has one **leader** replica and zero or more **follower** replicas on different brokers. All reads and writes go through the leader. Followers replicate by fetching from the leader.

The **In-Sync Replica (ISR)** set contains the leader and all followers that are "caught up" within a configurable lag threshold. When a producer sends a message with `acks=all`, the broker waits until all ISR members have acknowledged the write before confirming to the producer. If a follower falls behind, it is removed from the ISR. If the leader fails, a new leader is elected from the ISR.

This design gives you a tunable trade-off between durability and latency. `acks=all` with `min.insync.replicas=2` on a replication factor 3 topic means you can lose one broker without data loss and without downtime. Losing two brokers loses the partition (it becomes unavailable, not corrupted — assuming `unclean.leader.election.enable=false`, which it should be).

### ZooKeeper and KRaft

Historically, Kafka depended on **Apache ZooKeeper** for cluster metadata management: broker registration, controller election, topic configuration, and partition assignments. ZooKeeper worked, but it was a separate distributed system with its own operational requirements, failure modes, and scaling limits. Running ZooKeeper well is its own skillset, and many Kafka operational issues were actually ZooKeeper issues.

**KRaft** (Kafka Raft) is the long-awaited replacement, moving metadata management into the Kafka brokers themselves using a Raft-based consensus protocol. KRaft was marked production-ready in Kafka 3.3 (late 2022), and ZooKeeper support was formally deprecated. As of Kafka 4.0 (early 2025), new clusters should use KRaft exclusively. Migration from ZooKeeper to KRaft is supported but non-trivial — it involves running both systems in parallel during the transition.

KRaft eliminates the ZooKeeper dependency, simplifies deployment, and removes the metadata scaling bottleneck that limited Kafka clusters to hundreds of thousands of partitions. It also removes one of the most common "I set up Kafka in 10 minutes" blog post lies, since those 10 minutes never included ZooKeeper tuning.

---

## Producer Semantics

### The Basics

A Kafka producer sends records to topics. Each record has a key (optional), a value, a timestamp, and optional headers. The key determines partition assignment: records with the same key go to the same partition (assuming the partition count does not change), giving you per-key ordering.

### Acknowledgment Modes

The `acks` configuration controls durability:

- **`acks=0`**: Fire and forget. The producer does not wait for any acknowledgment. Maximum throughput, maximum data loss potential.
- **`acks=1`**: The leader writes to its local log and acknowledges. If the leader crashes before followers replicate, the message is lost. This is the default, which is a somewhat aggressive choice.
- **`acks=all` (or `acks=-1`)**: The leader waits for all ISR members to replicate before acknowledging. Combined with `min.insync.replicas=2`, this is the safe production setting.

### Idempotent Producer

Enabling `enable.idempotence=true` (the default since Kafka 3.0) assigns each producer a unique ID and sequence number per partition. The broker deduplicates messages with the same producer ID and sequence number. This prevents duplicates caused by producer retries — if the producer sends a message, the broker receives it but the acknowledgment is lost, the producer retries, and the broker recognises the duplicate.

Idempotent production is free in terms of configuration and nearly free in terms of performance. There is no good reason to disable it.

### Transactional Producer

For atomically writing to multiple partitions and topics — "either all of these messages are committed or none of them are" — Kafka provides transactional producers. This is the foundation of Kafka's exactly-once semantics (EOS).

The transactional producer coordinates with a **transaction coordinator** (a broker) to begin transactions, send messages, and commit or abort atomically. Combined with the idempotent producer and transactional consumers (using `read_committed` isolation), this provides exactly-once semantics within the Kafka ecosystem.

The catch: exactly-once applies to Kafka-to-Kafka pipelines. The moment you write to an external system (a database, an API), you are outside the transaction boundary. You need your own deduplication or two-phase commit mechanism for end-to-end exactly-once.

---

## Consumer Groups and Rebalancing

### Consumer Groups

Consumers are organised into **consumer groups**. Each partition in a topic is assigned to exactly one consumer in a group. If you have 6 partitions and 3 consumers in a group, each consumer gets 2 partitions. If you have 6 partitions and 8 consumers, 2 consumers sit idle. If you have 6 partitions and 1 consumer, that consumer handles all 6.

This is simple, elegant, and the source of a great deal of operational misery.

### Rebalancing

When a consumer joins or leaves a group — by starting up, crashing, or failing to send a heartbeat within `session.timeout.ms` — the group **rebalances**. During a rebalance, all consumers in the group stop processing, partitions are redistributed, and consumers resume from their last committed offset.

The problem: rebalancing is **stop-the-world**. For the duration of the rebalance, no messages are processed. In a large consumer group with many partitions, rebalancing can take seconds to minutes. If your consumers are slow to start, it takes even longer. If a consumer is flapping (repeatedly joining and leaving), you get rebalance storms — a cascade of rebalances that can effectively halt processing.

#### Mitigation Strategies

- **Static group membership** (`group.instance.id`): Assigns a persistent identity to each consumer so that temporary disconnections do not trigger rebalances.
- **Cooperative rebalancing** (`CooperativeStickyAssignor`): Instead of stop-the-world, only the affected partitions are revoked and reassigned. This dramatically reduces rebalance impact.
- **Incremental rebalancing** (Kafka 3.x+): Further improvements to minimize disruption.

These mitigations work well but require configuration. The default rebalancing behaviour is the stop-the-world "eager" protocol, because Kafka respects backward compatibility more than your uptime.

### Partition Assignment Strategies

- **RangeAssignor**: Assigns contiguous partition ranges to consumers. Can be uneven.
- **RoundRobinAssignor**: Distributes partitions evenly across consumers.
- **StickyAssignor**: Tries to minimize partition movement during rebalances.
- **CooperativeStickyAssignor**: Sticky + cooperative rebalancing. This is what you want.

---

## The Kafka Ecosystem

### Kafka Streams

A Java library for building stream processing applications. Not a separate cluster — it runs inside your application. Kafka Streams provides stateful operations (aggregations, joins, windowing) backed by local state stores (RocksDB) with changelog topics for fault tolerance.

Kafka Streams is genuinely excellent for Kafka-centric stream processing. It is also Java-only, which limits its audience. If your team writes Python or Go, Kafka Streams is not an option unless you want to maintain a separate Java service.

### ksqlDB

SQL-like syntax on top of Kafka Streams. Write `SELECT * FROM orders WHERE amount > 1000 EMIT CHANGES` and get a streaming query. Powerful for prototyping and simple transformations. Less suitable for complex business logic. It is a Confluent product, and its licensing has shifted over time — check the current terms before building on it.

### Kafka Connect

A framework for streaming data between Kafka and external systems using pre-built **connectors**. Source connectors pull data into Kafka (e.g., from a database via CDC). Sink connectors push data from Kafka to external systems (e.g., to Elasticsearch, S3, a data warehouse).

The connector ecosystem is Kafka's most underrated asset. There are hundreds of connectors — some from Confluent, some from the community, some from vendors. The quality varies, but the top-tier connectors (Debezium for CDC, S3 sink, JDBC source/sink) are production-grade and save enormous amounts of custom integration code.

Connect runs as a separate cluster of worker nodes, which means it is another thing to deploy, monitor, and scale. But the alternative — writing and maintaining custom producer/consumer code for every data integration — is worse.

### Schema Registry

Confluent Schema Registry stores Avro, Protobuf, and JSON Schema definitions and enforces compatibility rules (backward, forward, full) when schemas evolve. Producers and consumers negotiate schemas via the registry, and serialisation/deserialisation happens automatically.

Schema Registry is not part of Apache Kafka — it is a Confluent project. There is an open-source version under the Confluent Community License (not Apache 2.0) and alternatives like Apicurio Registry and AWS Glue Schema Registry. Schema management is essential for any serious Kafka deployment; which registry you use is a practical choice.

---

## Strengths

### Throughput

Kafka was built for throughput. A well-tuned cluster on modern hardware can sustain millions of messages per second. The sequential I/O design, zero-copy transfer (using `sendfile` to stream data directly from page cache to network socket without copying through the JVM), and batching at every layer make it extraordinarily efficient at moving large volumes of data.

### Ecosystem

Nothing else comes close. Client libraries in every mainstream language. Hundreds of connectors. Kafka Streams, ksqlDB, Schema Registry. Integration with every major data tool, cloud platform, and monitoring system. If you choose Kafka, you will rarely be the first person to solve a particular integration problem.

### Battle-Tested at Scale

Kafka runs at LinkedIn (7 trillion messages per day as of their last public disclosure), Netflix, Uber, Apple, and thousands of other companies. It has been hammered, broken, patched, and hardened by the most demanding workloads on the planet. The failure modes are well-documented. The operational practices are well-established. There is a large community of experienced operators.

### The Log Abstraction

The commit log model is simply a better abstraction than the traditional message queue for many workloads. Replay capability, consumer group independence, and time-based retention make it natural for event sourcing, stream processing, CDC, and analytics pipelines.

### Exactly-Once Semantics

Kafka is one of the few systems that offers genuine exactly-once semantics (within the Kafka ecosystem). The combination of idempotent producers, transactional writes, and consumer offset commits inside transactions is well-engineered and works reliably.

---

## Weaknesses

### Operational Complexity

Running Kafka well is a full-time job. A medium-sized deployment requires attention to broker configuration (there are over 200 configuration parameters), partition management, replication monitoring, consumer group health, disk management, JVM tuning, and network configuration. The learning curve from "it works on my laptop" to "it is reliable in production" is steep and expensive.

### JVM Tuning

Kafka runs on the JVM, and GC pauses are a real concern. Long GC pauses can cause brokers to drop out of the ISR, trigger leader elections, and increase tail latency. Tuning the garbage collector (G1 or ZGC, heap sizing, GC logging) is part of every serious Kafka deployment. The ZGC garbage collector in modern JVMs has improved things significantly, but it remains a factor.

### Rebalance Storms

Consumer group rebalancing, as discussed above, is Kafka's most annoying operational issue. It has improved dramatically with cooperative rebalancing and static membership, but legacy consumers using the default eager protocol will still experience it.

### Partition Management

Partitions are Kafka's unit of parallelism, but they are also its unit of operational pain. Each partition has a leader, followers, and metadata. More partitions means more metadata, more file handles, more recovery time after a broker failure, and more complex rebalancing. There is a practical upper limit to partition count per broker (historically tens of thousands, improved with KRaft), and getting the partition count wrong at topic creation is a mistake that haunts you forever — or at least until you recreate the topic.

### No Built-In Message Routing

Kafka has topics and partitions. If you want to route messages based on content (this order goes to the fraud detector, that order goes to the warehouse), you build that routing logic in your producers or stream processors. There is no equivalent to RabbitMQ's exchange-based routing. This is by design — Kafka is a log, not a router — but it means more application-level code for routing-heavy workloads.

### Cost at Scale

Kafka clusters need fast disks (SSDs for latency-sensitive workloads), plenty of memory (page cache is critical for performance), and enough network bandwidth for replication and consumer traffic. A three-broker cluster with replication factor 3, reasonable retention, and production-grade monitoring is not cheap. It is cheaper than the alternatives at very high throughput, but it is expensive at low to moderate throughput.

---

## Ideal Use Cases

- **High-throughput event streaming**: clickstream, IoT telemetry, log aggregation, metrics pipelines
- **Event sourcing and CQRS**: the log is a natural event store
- **Stream processing**: when combined with Kafka Streams, ksqlDB, or Flink
- **Change data capture**: with Debezium and Kafka Connect
- **Data integration hub**: centralized pipeline between operational and analytical systems
- **Microservice event bus** (at scale): when you have enough traffic to justify the operational overhead

---

## Operational Reality

### Minimum Viable Cluster

Three brokers with KRaft (three controller nodes, which can be co-located with brokers for small clusters). Replication factor 3, `min.insync.replicas=2`. This gives you single-node failure tolerance. For development, a single broker works, but do not mistake development for production.

### Key Monitoring Metrics

- **Under-replicated partitions**: Any value above 0 is a red alert. It means data is at risk.
- **Consumer lag**: How far behind are your consumers? Growing lag means consumers cannot keep up.
- **Request latency (produce, fetch)**: p99 latency increasing? Time to investigate.
- **ISR shrink/expand rate**: Frequent ISR changes indicate broker instability.
- **Controller metrics**: Leader elections, active controller count.
- **Disk usage and I/O**: Kafka is I/O-bound. Watch for disk saturation.
- **JVM GC pauses**: Long pauses directly impact broker responsiveness.

Use Prometheus with the JMX Exporter, plus Grafana dashboards. There are well-established community dashboards. Confluent Control Center provides a richer view but requires a Confluent licence.

### Upgrades

Kafka supports rolling upgrades with zero downtime, but the process requires care:

1. Update broker configurations for the new inter-broker protocol version.
2. Roll brokers one at a time, waiting for each to rejoin the ISR before proceeding.
3. Update the inter-broker protocol version cluster-wide.
4. Update the log message format version.

Client library versions must be compatible with the broker version. Kafka maintains backward compatibility for several major versions, but testing is essential.

### Multi-Datacenter

Kafka is not natively multi-datacenter. Options:

- **MirrorMaker 2**: Replicates topics between clusters. It works, it is asynchronous (so some data loss during failover is possible), and managing the replication topology requires attention.
- **Confluent Replicator**: A commercial alternative with more features.
- **Stretched clusters**: Running a single cluster across datacenters with rack-aware replica placement. This works but requires low-latency inter-datacenter links and careful configuration. Latency between DCs directly impacts produce latency with `acks=all`.

### Managed Offerings

- **Confluent Cloud**: The most feature-rich managed Kafka. Serverless and dedicated options. Schema Registry, ksqlDB, Connect managed. Not cheap, but eliminates operational burden.
- **Amazon MSK**: AWS-managed Kafka. Less opinionated, gives you the raw brokers. MSK Serverless is simpler but has limitations. You still manage topics, consumers, and monitoring.
- **Aiven for Apache Kafka**: Multi-cloud managed Kafka with a clean interface. Good support for open-source tooling.
- **Azure Event Hubs (Kafka-compatible)**: Not actually Kafka, but implements the Kafka protocol. Works for basic use cases; do not expect full Kafka feature parity.
- **Redpanda Cloud**: A Kafka-compatible alternative, not Apache Kafka. Covered in its own chapter.

---

## Code Examples

### Java Producer

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;

public class OrderEventProducer {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

        // Production settings — do not skip these
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
        props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);

        try (KafkaProducer<String, String> producer = new KafkaProducer<>(props)) {
            ProducerRecord<String, String> record = new ProducerRecord<>(
                "order-events",          // topic
                "order-7829",            // key (partition routing)
                "{\"type\":\"OrderPlaced\",\"orderId\":\"order-7829\"}"
            );

            producer.send(record, (metadata, exception) -> {
                if (exception != null) {
                    System.err.println("Send failed: " + exception.getMessage());
                } else {
                    System.out.printf("Sent to partition %d at offset %d%n",
                        metadata.partition(), metadata.offset());
                }
            });
        }
    }
}
```

### Java Consumer

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.List;
import java.util.Properties;

public class OrderEventConsumer {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processing-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);

        // Start from earliest offset if no committed offset exists
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        // Disable auto-commit — commit manually after processing
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        // Use cooperative rebalancing to avoid stop-the-world rebalances
        props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
            CooperativeStickyAssignor.class.getName());

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(List.of("order-events"));

            while (true) {
                ConsumerRecords<String, String> records =
                    consumer.poll(Duration.ofMillis(100));

                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("Received: key=%s, value=%s, partition=%d, offset=%d%n",
                        record.key(), record.value(),
                        record.partition(), record.offset());

                    // Process the record here...
                }

                // Commit offsets after processing the batch
                consumer.commitSync();
            }
        }
    }
}
```

### Python Producer (confluent-kafka)

```python
from confluent_kafka import Producer

conf = {
    'bootstrap.servers': 'localhost:9092',
    'acks': 'all',
    'enable.idempotence': True,
    'retries': 10000000,
}

producer = Producer(conf)

def delivery_callback(err, msg):
    if err:
        print(f'Delivery failed: {err}')
    else:
        print(f'Delivered to {msg.topic()} [{msg.partition()}] @ {msg.offset()}')

producer.produce(
    topic='order-events',
    key='order-7829',
    value='{"type": "OrderPlaced", "orderId": "order-7829"}',
    callback=delivery_callback,
)

# flush() blocks until all messages are delivered or timeout
producer.flush(timeout=10)
```

### Python Consumer (confluent-kafka)

```python
from confluent_kafka import Consumer

conf = {
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'order-processing-group',
    'auto.offset.reset': 'earliest',
    'enable.auto.commit': False,
}

consumer = Consumer(conf)
consumer.subscribe(['order-events'])

try:
    while True:
        msg = consumer.poll(timeout=1.0)
        if msg is None:
            continue
        if msg.error():
            print(f'Consumer error: {msg.error()}')
            continue

        print(f'Received: key={msg.key()}, value={msg.value()}, '
              f'partition={msg.partition()}, offset={msg.offset()}')

        # Process the message here...

        # Commit the offset after successful processing
        consumer.commit(asynchronous=False)
finally:
    consumer.close()
```

---

## Verdict

Kafka is the obvious choice for high-throughput event streaming — if you can afford its operational cost. "Afford" here means both money and expertise. If your organisation has a platform team that can dedicate time to Kafka operations, or if you are willing to pay for a managed service, Kafka's ecosystem, throughput, and battle-tested reliability make it the strongest option for data-intensive workloads.

**Pick Kafka when:**
- You need sustained high throughput (hundreds of thousands to millions of messages per second)
- You need event replay and stream processing
- You need a rich connector ecosystem for data integration
- You have the team (or the budget for managed services) to operate it properly
- You are building a centralised event streaming platform for multiple teams

**Avoid Kafka when:**
- Your throughput is modest and a simpler broker would suffice
- You need complex message routing (look at RabbitMQ)
- You are a small team without dedicated infrastructure engineers and do not want to pay for managed Kafka
- Your use case is simple task queues or request-reply patterns
- You need native multi-tenancy with strong isolation (look at Pulsar)

Kafka is not the answer to every messaging problem. But for the problems it was designed to solve — high-volume, durable, replayable event streaming — it remains the benchmark against which everything else is measured.
