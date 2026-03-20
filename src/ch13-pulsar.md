# Apache Pulsar

Apache Pulsar is what happens when you look at Kafka and think, "What if we separated the compute from the storage, added multi-tenancy from day one, made geo-replication a first-class feature, and accepted that the result would be three distributed systems in a trenchcoat?" The answer is a platform that is genuinely impressive in its capabilities and genuinely demanding in its operational requirements.

Pulsar occupies a fascinating position in the messaging landscape. It addresses real limitations of Kafka — multi-tenancy, tiered storage, geo-replication, and the unified queuing-plus-streaming model — with architectural choices that are technically sound but operationally expensive. Whether those capabilities justify that expense depends entirely on your specific needs. This chapter will help you figure out if you are one of the organisations for whom Pulsar is the right answer.

---

## Overview

### What It Is

Apache Pulsar is a cloud-native, distributed messaging and event streaming platform. It provides pub/sub messaging, event streaming with replay, and traditional message queuing — all within a single system. Its distinguishing architectural feature is the separation of serving (brokers) and storage (Apache BookKeeper), which enables independent scaling of compute and storage.

### Brief History

Pulsar was created at Yahoo! in 2013. Yahoo! needed a messaging platform that could serve multiple business units (Yahoo Mail, Yahoo Finance, Flickr, Tumblr) on shared infrastructure without one team's misbehaving workload taking down another team's production service. Nothing on the market did this. Kafka had no meaningful multi-tenancy. RabbitMQ was not designed for Yahoo!'s scale. So they built their own.

The key design decisions were made early: separate compute from storage, build multi-tenancy into the core rather than bolting it on later, and support both queuing and streaming semantics. The implementation leveraged Apache BookKeeper — a distributed, write-ahead log storage system originally developed at Yahoo! for HDFS NameNode journaling — as the durable storage layer.

Yahoo! open-sourced Pulsar in 2016 and donated it to the Apache Software Foundation, where it became a Top-Level Project in 2018. StreamNative, founded by members of the original Yahoo! Pulsar team (including Sijie Guo and Jia Zhai), became the primary commercial entity behind Pulsar, offering managed services and enterprise features.

The project's journey has not been entirely smooth. Community governance disputes, competition with the well-funded Confluent ecosystem, and the inherent complexity of the system have all created headwinds. The community is smaller than Kafka's but dedicated, and development continues actively.

### Who Runs It

The Apache Software Foundation governs the project. StreamNative employs many of the core committers. DataStax (through its Astra Streaming offering) also contributes to the ecosystem. The project has a more diverse committer base than Kafka (which is dominated by Confluent employees), though the total contributor count is smaller.

---

## Architecture

### The Two-Layer Architecture

This is the fundamental difference between Pulsar and Kafka, and it cascades into everything else.

**Kafka's architecture**: Brokers serve requests *and* store data. A Kafka broker is a stateful node — it owns partitions on its local disks. Scaling storage means adding brokers. Rebalancing data means moving partitions between brokers, which is an expensive, bandwidth-intensive operation.

**Pulsar's architecture**: Brokers serve requests. BookKeeper bookies store data. The two layers are independent.

**Pulsar Brokers** are stateless (mostly — they cache data, but the source of truth is in BookKeeper). They handle producer and consumer connections, manage topic ownership, and serve read/write requests. Because they are stateless, they can be added, removed, and replaced without data movement. Scaling the serving layer is fast and minimally disruptive.

**Apache BookKeeper Bookies** are the storage nodes. Each bookie manages a set of ledgers (sequential log segments). When a Pulsar topic receives messages, they are written to a set of bookies in parallel. Bookies are stateful, but their data is managed in smaller units (ledgers/segments) than Kafka's partitions, which makes storage operations more granular.

**ZooKeeper** (yes, ZooKeeper again) manages metadata for both Pulsar and BookKeeper: broker registration, topic ownership, ledger metadata, and configuration. Pulsar's ZooKeeper dependency is more extensive than Kafka's was, because both the broker layer and the storage layer rely on it.

### Segments vs Partitions

This is a nuance worth understanding. In Kafka, a partition is a single log file (or set of segment files) on a single broker. The entire partition lives on one broker (plus its replicas). This means:

- The maximum size of a partition is limited by the broker's local disk.
- Rebalancing a partition means moving the entire thing to another broker.
- If a broker fails, its partitions are unavailable until a new leader is elected from the replicas.

In Pulsar, a topic partition is divided into **segments** (BookKeeper ledgers). Each segment is stored across multiple bookies (striped, not residing on a single node). When the current segment reaches a size or time threshold, a new segment is created on potentially different bookies. This means:

- **No single-broker storage bottleneck**: A topic's data is distributed across many bookies.
- **Faster recovery**: When a bookie fails, only its segments need to be recovered, and the recovery reads come from multiple surviving bookies in parallel.
- **Tiered storage**: Old segments can be offloaded to cheap object storage (S3, GCS, Azure Blob) while hot segments remain on bookies. This is built into Pulsar's design, not an afterthought.

The downside: this architecture means more components, more metadata, more coordination, and more things that can go wrong.

### Tiered Storage

Pulsar's tiered storage model is one of its most compelling features for cost-conscious, high-retention workloads.

Messages flow through three tiers:

1. **BookKeeper (hot storage)**: Recent messages on bookie disks (SSD or HDD). Fast read/write access.
2. **Object storage (cold storage)**: Older segments offloaded to S3, GCS, or Azure Blob Storage. Dramatically cheaper per GB.
3. **(Optional) Local cache on brokers**: Frequently accessed data cached in broker memory for low-latency reads.

This means you can retain months or years of event history at object storage prices, while recent data remains on fast storage for low-latency access. Kafka can do this too (with Confluent's Tiered Storage or community implementations), but Pulsar had it first and it is more deeply integrated.

---

## Multi-Tenancy

Multi-tenancy is Pulsar's headline feature and the reason it was built.

### The Hierarchy

Pulsar organises resources in a three-level hierarchy:

1. **Tenants**: Top-level organisational unit. Typically maps to a team, business unit, or application. Each tenant has its own admin permissions and resource policies.
2. **Namespaces**: A grouping of topics within a tenant. Policies (retention, backlog limits, replication, schema enforcement) are set at the namespace level.
3. **Topics**: The actual message streams, living within a namespace.

The full topic name looks like: `persistent://tenant/namespace/topic-name`

### Isolation Mechanisms

- **Authentication and authorisation**: Per-tenant access control. One team cannot access another team's topics without explicit permission.
- **Resource quotas**: Limits on message rate, bandwidth, storage, and number of topics per namespace or tenant.
- **Namespace isolation policies**: You can designate specific brokers for specific namespaces, ensuring that a noisy tenant's traffic does not compete with a latency-sensitive tenant's brokers.
- **Backlog quotas**: Limits on how much unconsumed data a topic can accumulate before producers are throttled or the oldest data is dropped.

This is genuinely more sophisticated than what Kafka or RabbitMQ offer. Kafka has ACLs and quotas, but no formal tenant abstraction. RabbitMQ has vhosts, which provide isolation but not the policy richness of Pulsar's namespace system.

---

## Geo-Replication

Pulsar's geo-replication is built into the core, not bolted on as an external tool.

### How It Works

Configure two (or more) Pulsar clusters in different regions. Set a replication policy on a namespace. Pulsar's brokers automatically replicate messages between clusters, using dedicated replication connections. Each cluster maintains its own copy of the data with its own consumer offsets.

Replication is asynchronous (so there is a lag window during which a disaster in one region can lose the most recent messages), but the mechanism is integrated, monitored via standard Pulsar metrics, and configured through the standard admin API.

### Comparison

- **Kafka**: Requires MirrorMaker 2 or Confluent Replicator — separate processes that consume from one cluster and produce to another. It works, but it is more operational moving parts.
- **RabbitMQ**: Federation and shovel provide cross-cluster replication, but with less sophistication and no namespace-level policy control.
- **Pulsar**: Built-in, policy-driven, per-namespace. The cleanest implementation among the three.

---

## Subscription Types

Pulsar supports four subscription types, giving you more consumer patterns than either Kafka or RabbitMQ in a single system.

**Exclusive**: One consumer on the subscription. If another consumer tries to subscribe, it is rejected. This is the simplest and provides strict ordering.

**Shared**: Multiple consumers share the subscription. Messages are distributed round-robin across consumers. This is the traditional work queue pattern. Ordering is not guaranteed because different consumers process messages at different speeds.

**Failover**: One active consumer, one or more standby consumers. If the active consumer disconnects, a standby takes over. Ordering is preserved (within a partition) during normal operation.

**Key_Shared**: Messages with the same key are delivered to the same consumer, while messages with different keys can be distributed across consumers. This gives you per-key ordering with parallelism across keys — the same pattern as Kafka's partition-key-based consumer model, but without requiring you to pre-configure partition counts.

The availability of all four subscription types on the same topic is a significant advantage. In Kafka, you get exclusive-per-partition (roughly equivalent to failover) and that is it — shared consumption requires architectural workarounds. In RabbitMQ, you get shared consumption from queues but not the Kafka-style exclusive-per-partition model (without manual coordination).

---

## Pulsar Functions and Pulsar IO

### Pulsar Functions

Lightweight, serverless-style compute functions that run inside the Pulsar cluster. A Pulsar Function consumes messages from one or more topics, processes them, and produces results to another topic. Supported languages: Java, Python, Go.

Pulsar Functions are useful for simple transformations, routing, and enrichment — the kind of glue logic that does not justify a separate stream processing framework. They run as threads within the broker, in separate processes, or in Kubernetes pods.

They are not a replacement for Kafka Streams or Flink. For complex stateful stream processing (windowed aggregations, multi-way joins, exactly-once stateful operations), you need an external framework. But for lightweight processing, they reduce the operational surface area by keeping the logic close to the broker.

### Pulsar IO

Pulsar's equivalent of Kafka Connect. Source connectors pull data into Pulsar; sink connectors push data from Pulsar to external systems. The connector ecosystem is smaller than Kafka Connect's — significantly smaller. This is a meaningful practical limitation.

Available connectors cover the common cases (JDBC, Elasticsearch, Cassandra, Kafka adapter, S3), but the long tail of specialised connectors that Kafka Connect offers is not there. If your integration needs are standard, Pulsar IO works. If you need a connector for an obscure system, you may be writing it yourself.

---

## Schema Registry

Pulsar includes a schema registry as a built-in feature — no separate service to deploy.

Schemas are associated with topics. Producers declare their schema when connecting, and the broker validates messages against the registered schema. Schema compatibility is enforced (backward, forward, full, or none). Supported formats include Avro, Protobuf, JSON Schema, and primitive types.

Having the schema registry built in (rather than as a separate service like Confluent Schema Registry) simplifies deployment and ensures that schema enforcement is always available. The trade-off is that Pulsar's schema registry is less feature-rich than Confluent's — fewer compatibility modes, less tooling around it.

---

## Strengths

### Multi-Tenancy

The best multi-tenancy implementation in the open-source messaging world. If you are building a shared messaging platform for multiple teams with different requirements, Pulsar handles this natively where other brokers require operational gymnastics or multiple clusters.

### Geo-Replication

Built-in, policy-driven, per-namespace. Simpler to configure and operate than Kafka's MirrorMaker 2 and more capable than RabbitMQ's federation.

### Tiered Storage

Native support for offloading old data to object storage. This makes Pulsar significantly cheaper for high-retention workloads compared to keeping everything on broker-local SSDs.

### Unified Messaging Model

Queuing and streaming from the same platform, with four subscription types. You do not need to run RabbitMQ for your work queues and Kafka for your event streams — Pulsar can do both. Whether the operational complexity of Pulsar is less than the operational complexity of two separate systems is a calculation worth doing carefully.

### Scalability Architecture

The separation of brokers and bookies means you can scale serving and storage independently. Adding read capacity (more brokers) does not require moving data. Adding storage capacity (more bookies) does not require migrating topic ownership. This is architecturally elegant and practically valuable at large scale.

### Built-In Schema Registry

One less component to deploy and manage. Schema enforcement is always available.

---

## Weaknesses

### Three Distributed Systems in a Trenchcoat

This is the elephant in the room. A Pulsar deployment consists of:

1. **Pulsar brokers** — a distributed, stateful (caching) service.
2. **Apache BookKeeper bookies** — a distributed storage system with its own replication, journaling, and garbage collection.
3. **Apache ZooKeeper** — a distributed coordination service.

Each of these is a production system that needs to be deployed, configured, monitored, scaled, and upgraded. Each has its own failure modes, its own performance characteristics, and its own operational expertise requirements.

When advocates say Pulsar is "more complex" than Kafka, this is what they mean. It is not that any single component is harder than a Kafka broker. It is that you are operating three production distributed systems instead of one (or two, if you count Kafka's now-deprecated ZooKeeper dependency). The total operational surface area is larger.

With KRaft, Kafka has eliminated its ZooKeeper dependency. Pulsar has not — both its broker layer and its storage layer depend on ZooKeeper. There is ongoing work to reduce this dependency (using the upcoming Oxia metadata store as an alternative), but as of this writing, ZooKeeper remains a hard requirement.

### Smaller Ecosystem

Pulsar's client libraries are good for Java. They are adequate for Python, Go, and C++. They are less mature for other languages. The community-maintained clients vary in quality.

The connector ecosystem (Pulsar IO) is a fraction of Kafka Connect's. The stream processing options (Pulsar Functions) are lightweight compared to Kafka Streams. The tooling ecosystem is thinner — fewer monitoring dashboards, fewer management tools, fewer blog posts explaining how to solve specific problems.

This matters in practice. When you hit an obscure issue with Kafka, someone has probably blogged about it. When you hit an obscure issue with Pulsar, you may be reading the source code.

### BookKeeper Expertise

BookKeeper is a powerful storage system, but it is not widely known. The operational knowledge base is small. Tuning BookKeeper — journal device configuration, ledger device configuration, compaction settings, read/write quorum sizes — requires understanding its internals. Finding engineers with BookKeeper experience is harder than finding engineers with Kafka experience. Significantly harder.

When a Pulsar cluster misbehaves, the root cause is often in BookKeeper. Diagnosing slow writes, compaction backlogs, or ledger metadata issues requires a different skillset than diagnosing Kafka partition issues.

### Community Size and Momentum

Pulsar's community is active but smaller than Kafka's. Fewer contributors means slower bug fixes for non-critical issues, fewer third-party integrations, and a smaller pool of operational knowledge. The project has had some governance turbulence — disputes between corporate contributors, concerns about the PMC composition — that have consumed energy that could have gone into development.

Pulsar is not at risk of abandonment. It is a viable, actively developed project. But the community tailwinds that Kafka enjoys are not there to the same degree.

### Learning Curve

The concept count is high. Tenants, namespaces, topics (persistent and non-persistent), partitioned topics, subscriptions (four types), cursors, ledgers, segments, bookies, brokers, functions, IO connectors. A new engineer needs to understand more concepts before they can operate Pulsar confidently than they would for Kafka or RabbitMQ.

---

## Ideal Use Cases

- **Multi-tenant messaging platforms**: The use case Pulsar was designed for. Shared infrastructure for multiple teams with isolation and per-tenant policies.
- **Global, geo-replicated messaging**: When you need active-active messaging across regions with built-in replication.
- **High-retention event streaming**: When you need months or years of retention without paying SSD prices for cold data.
- **Mixed workloads**: When the same platform needs to handle both event streaming (Kafka-style) and traditional queuing (RabbitMQ-style).
- **Large-scale deployments**: Where the ability to scale brokers and storage independently provides meaningful operational advantages.

---

## Operational Reality

### Minimum Viable Cluster

A production Pulsar deployment requires, at minimum:

- **3 ZooKeeper nodes** (for metadata)
- **3 BookKeeper bookies** (for data storage, with write quorum 3, ack quorum 2)
- **2+ Pulsar brokers** (for serving, stateless so you want at least 2 for availability)

That is 8 nodes minimum. Compare this to Kafka's 3 nodes (with KRaft). The infrastructure cost of entry is higher.

For development and testing, you can run everything on a single machine with Pulsar's standalone mode, which bundles a broker, bookie, and ZooKeeper in one process. Do not confuse standalone mode with production readiness.

### Key Monitoring Metrics

**Broker metrics:**
- Throughput (messages in/out per topic, namespace, tenant)
- Publish latency (p50, p99)
- Subscription backlog (messages and bytes)
- Topic count and memory usage
- Connection count

**BookKeeper metrics:**
- Write latency (journal and ledger)
- Read latency
- Disk usage and I/O per bookie
- Compaction progress (garbage collection of deleted data)
- Ledger count and open ledger count

**ZooKeeper metrics:**
- Request latency
- Outstanding requests
- Connection count
- Leader election events

You need monitoring for all three layers. A Pulsar cluster is only as healthy as its least healthy component. BookKeeper degradation is often the first sign of trouble and the hardest to diagnose.

Prometheus metrics are available for all components, and there are community Grafana dashboards. StreamNative offers enhanced monitoring through their commercial tools.

### Upgrade Strategy

Upgrading Pulsar involves upgrading three systems, in order:

1. **ZooKeeper** (if required by the new version)
2. **BookKeeper bookies** (rolling upgrade, one at a time, waiting for replication to catch up)
3. **Pulsar brokers** (rolling upgrade, simpler because they are quasi-stateless)

The ordering matters — brokers depend on bookies, and both depend on ZooKeeper. You cannot upgrade brokers first.

Protocol compatibility is generally maintained within minor versions, but major version upgrades require careful testing. The documentation for upgrade paths is adequate but not as exhaustive as Kafka's.

### The ZooKeeper + BookKeeper Dependency Chain

This deserves its own section because it is the source of most Pulsar operational pain.

ZooKeeper is a mature system, but it is sensitive to disk latency (it writes to a transaction log synchronously) and has a well-known throughput limit on the number of writes per second. In a large Pulsar deployment, the metadata operations from both brokers and bookies can push ZooKeeper to its limits. Symptoms include increasing request latency, session timeouts, and — in the worst case — ZooKeeper ensemble instability that cascades into both broker and bookie failures.

BookKeeper's dependency on ZooKeeper for ledger metadata adds another failure coupling. If ZooKeeper becomes slow, BookKeeper cannot create new ledgers, which means Pulsar cannot roll segments, which means writes eventually stall.

Mitigations:
- Dedicated ZooKeeper nodes (do not co-locate with brokers or bookies in production)
- Fast SSDs for ZooKeeper transaction logs
- Separate ZooKeeper ensembles for Pulsar and BookKeeper (adds more nodes but isolates failure domains)
- Monitor ZooKeeper latency obsessively

---

## Managed Offerings

- **StreamNative Cloud**: The primary managed Pulsar offering, from the team that built it. Fully managed, multi-cloud (AWS, GCP, Azure). Includes management UI, monitoring, and support. This is the easiest way to run Pulsar if you want the capabilities without the operational burden.
- **DataStax Astra Streaming**: Pulsar-based managed service, part of the DataStax Astra platform. Positioned alongside Astra DB (Cassandra). The Pulsar integration is solid, and the pricing model is consumption-based.

The managed offerings are the honest recommendation for most teams considering Pulsar. The operational complexity of self-managed Pulsar is high enough that the managed service premium is often worth it — unless you have a dedicated platform team with distributed systems expertise and the specific need for self-hosted infrastructure.

---

## Code Examples

### Java Producer

```java
import org.apache.pulsar.client.api.*;

public class OrderEventProducer {
    public static void main(String[] args) throws PulsarClientException {
        PulsarClient client = PulsarClient.builder()
            .serviceUrl("pulsar://localhost:6650")
            .build();

        Producer<String> producer = client.newProducer(Schema.STRING)
            .topic("persistent://public/default/order-events")
            .producerName("order-service")
            .sendTimeout(10, java.util.concurrent.TimeUnit.SECONDS)
            .blockIfQueueFull(true)
            .create();

        try {
            MessageId messageId = producer.newMessage()
                .key("order-7829")
                .value("{\"type\":\"OrderPlaced\",\"orderId\":\"order-7829\"}")
                .property("region", "us-east")
                .send();

            System.out.println("Published message: " + messageId);
        } finally {
            producer.close();
            client.close();
        }
    }
}
```

### Java Consumer (Shared Subscription)

```java
import org.apache.pulsar.client.api.*;

public class OrderEventConsumer {
    public static void main(String[] args) throws PulsarClientException {
        PulsarClient client = PulsarClient.builder()
            .serviceUrl("pulsar://localhost:6650")
            .build();

        Consumer<String> consumer = client.newConsumer(Schema.STRING)
            .topic("persistent://public/default/order-events")
            .subscriptionName("order-processing")
            .subscriptionType(SubscriptionType.Key_Shared)  // Per-key ordering
            .ackTimeout(30, java.util.concurrent.TimeUnit.SECONDS)
            .subscribe();

        try {
            while (true) {
                Message<String> msg = consumer.receive();

                try {
                    System.out.printf("Received: key=%s, value=%s, messageId=%s%n",
                        msg.getKey(), msg.getValue(), msg.getMessageId());

                    // Process the message...

                    // Acknowledge after successful processing
                    consumer.acknowledge(msg);

                } catch (Exception e) {
                    // Negative acknowledge — message will be redelivered
                    consumer.negativeAcknowledge(msg);
                }
            }
        } finally {
            consumer.close();
            client.close();
        }
    }
}
```

### Python Producer

```python
import pulsar

client = pulsar.Client('pulsar://localhost:6650')

producer = client.create_producer(
    topic='persistent://public/default/order-events',
    producer_name='order-service-python',
    block_if_queue_full=True,
)

message_id = producer.send(
    content='{"type": "OrderPlaced", "orderId": "order-7829"}'.encode('utf-8'),
    partition_key='order-7829',
    properties={'region': 'us-east'},
)

print(f'Published message: {message_id}')

producer.close()
client.close()
```

### Python Consumer

```python
import pulsar

client = pulsar.Client('pulsar://localhost:6650')

consumer = client.subscribe(
    topic='persistent://public/default/order-events',
    subscription_name='order-processing',
    consumer_type=pulsar.ConsumerType.KeyShared,
)

while True:
    msg = consumer.receive()
    try:
        data = msg.data().decode('utf-8')
        print(f'Received: key={msg.partition_key()}, value={data}')

        # Process the message...

        consumer.acknowledge(msg)

    except Exception as e:
        print(f'Processing failed: {e}')
        consumer.negative_acknowledge(msg)

consumer.close()
client.close()
```

---

## Verdict

Pulsar is the right choice for a specific set of problems, and for those problems, it is arguably the best option available. Multi-tenant messaging platforms, geo-replicated deployments, and high-retention workloads with tiered storage are areas where Pulsar's architecture provides genuine advantages over the alternatives.

The cost is complexity. You are operating three distributed systems. You need expertise that is harder to hire for. The ecosystem is thinner. The community is smaller. These are not theoretical concerns — they are the daily reality of running Pulsar in production.

**Pick Pulsar when:**
- Multi-tenancy is a hard requirement, not a nice-to-have
- You need built-in geo-replication across regions
- You need cost-effective long-term retention via tiered storage
- You want both queuing and streaming patterns from a single platform
- You have a platform team with distributed systems expertise (or you are using a managed service)
- You need to scale serving and storage independently

**Avoid Pulsar when:**
- Your deployment is single-tenant (Kafka is simpler and more mature for this)
- You do not need geo-replication or tiered storage (you are paying complexity tax for features you are not using)
- Your team is small and cannot absorb the operational overhead of three distributed systems
- You need a large connector ecosystem (Kafka Connect wins)
- You need extensive stream processing (Kafka Streams and Flink integration are more mature in the Kafka ecosystem)

Pulsar is not Kafka's replacement. It is Kafka's alternative for workloads that need what Kafka does not offer natively. If you need multi-tenancy or geo-replication badly enough, Pulsar earns its complexity. If you do not, you are choosing the harder path for no reason — which is not engineering, it is masochism.
