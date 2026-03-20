# The Comparison Matrix

You have now read detailed chapters on sixteen brokers and a collection of niche systems. You have absorbed thousands of words about throughput, latency, durability, delivery semantics, and operational complexity. If you are anything like most engineers at this stage, you are thinking: "Just give me the table." Fair enough.

This chapter is the table. Or rather, it is several tables, plus the caveats and context that prevent those tables from being actively misleading. Because here is the uncomfortable truth about comparison matrices: they are simultaneously the most requested and the most dangerous form of technical documentation. A table compresses nuanced, context-dependent, workload-specific trade-offs into neat cells that fit on a single screen. That compression is useful for orientation and terrible for decision-making. Use this chapter to narrow your shortlist. Do not use it to make your final choice.

Every value in the tables below is a simplification. "High throughput" means different things at different message sizes, replication factors, and durability settings. "Low operational complexity" means different things depending on whether your team has Kubernetes expertise, Erlang experience, or a JVM tuning fetish. Read the individual chapters for the context behind each rating. The table tells you *what*. The chapters tell you *why*.

---

## The Big Table

This is the primary comparison matrix, covering the evaluation dimensions from Chapter 10. Ratings are relative to the other brokers in this table, not absolute. A "Medium" throughput rating does not mean the broker is slow — it means other brokers in this comparison are faster under comparable conditions.

| Broker | Max Throughput | p99 Latency | Durability | Ordering | Delivery Semantics | Ops Complexity | Ecosystem | Cost Model | Cloud-Native | Multi-Tenancy |
|---|---|---|---|---|---|---|---|---|---|---|
| **Kafka** | Very High (millions msg/s, GBs/s with partitions) | Medium (2-15ms typical, GC spikes) | High (ISR replication, configurable acks) | Partition-level | At-least-once; exactly-once within Kafka | High (ZK/KRaft, partition management, tuning) | Very Large (Connect, Streams, Schema Registry, massive client ecosystem) | Open source + infra; managed (Confluent, MSK) per-partition/hr | Good (K8s operators, tiered storage, managed offerings) | Limited (ACLs, quotas, no native namespaces) |
| **RabbitMQ** | Medium (50-100K msg/s typical per node) | Low-Medium (sub-ms to low-ms for simple queues) | High (quorum queues with Raft) | Queue-level | At-least-once; at-most-once configurable | Medium (single binary, Erlang runtime, clustering can be finicky) | Large (many client libs, plugins, management UI, Shovel, Federation) | Open source + infra; managed (CloudAMQP) | Moderate (K8s operator exists, stateful nature fights K8s) | Good (vhosts, per-vhost permissions and policies) |
| **Pulsar** | Very High (comparable to Kafka with enough bookies) | Medium (similar to Kafka, BookKeeper adds latency) | Very High (BookKeeper, fencing, rack-aware replication) | Partition-level; key-shared for per-key | At-least-once; exactly-once (transactional) | Very High (ZK + BookKeeper + brokers, three systems to operate) | Medium-Large (clients in many languages, Pulsar Functions, IO connectors) | Open source + infra; managed (StreamNative) | Good (tiered storage, K8s operators) | Excellent (native tenants, namespaces, policies, quotas) |
| **SNS/SQS** | High (SQS: virtually unlimited with horizontal scaling) | Medium (SQS: 10-50ms typical, polling-based) | Very High (AWS-managed, multi-AZ by default) | FIFO queues: per-group; Standard: best-effort | At-least-once (standard); exactly-once (FIFO) | Very Low (fully managed, nothing to operate) | AWS-native (Lambda triggers, EventBridge integration, limited outside AWS) | Per-request + per-GB data transfer; can get expensive at high volume | Excellent (it *is* the cloud) | Limited (IAM-based, no native namespaces) |
| **EventBridge** | Medium (default quotas, request increases available) | Medium-High (50-500ms typical end-to-end) | Very High (AWS-managed, multi-AZ) | None guaranteed | At-least-once | Very Low (fully managed, rule-based routing) | AWS-native (deep integration with 100+ AWS services, SaaS partners) | Per-event; cheap at low volume, adds up at high volume | Excellent (serverless-native) | Limited (per-account event buses, cross-account sharing) |
| **Google Pub/Sub** | Very High (auto-scales, no partitioning to manage) | Medium (50-100ms typical) | Very High (synchronous replication across zones) | Ordering keys (per-key ordering) | At-least-once; exactly-once (per-subscription) | Very Low (fully managed) | GCP-native (Dataflow, BigQuery subscriptions, many client libs) | Per-message + per-GB egress; volume discounts | Excellent (it *is* the cloud) | Moderate (IAM, per-project isolation) |
| **Azure Event Hubs** | Very High (throughput units, Kafka wire-compatible) | Medium (comparable to Kafka) | Very High (Azure-managed, zone-redundant) | Partition-level (Kafka-compatible) | At-least-once | Low (managed, but throughput unit planning needed) | Azure-native (Stream Analytics, Functions, Kafka compatibility layer) | Per throughput unit/hr + per-event ingress | Excellent (native Azure integration) | Moderate (consumer groups, namespace-level isolation) |
| **Redis Streams** | High (100K+ msg/s per node easily) | Very Low (sub-ms to low single-digit ms) | Medium (AOF/RDB, Redis Cluster replication is async) | Stream-level (within a single stream) | At-least-once (with consumer groups and ACK) | Low-Medium (Redis is well-known, but clustering adds complexity) | Large as Redis, small as a streaming platform (limited connectors) | Open source + infra; managed (ElastiCache, Redis Cloud) | Good (well-supported on K8s, managed offerings) | Limited (database-level isolation, no native multi-tenancy) |
| **NATS/JetStream** | High (NATS core: millions msg/s; JetStream: lower with persistence) | Very Low (NATS core: sub-ms; JetStream: low-ms) | Medium-High (JetStream: Raft-based, R3 replication) | Stream-level; per-subject with consumers | At-least-once; exactly-once (double-ack) | Low (single binary, simple config, built-in monitoring) | Medium (growing client ecosystem, no equivalent to Kafka Connect) | Open source + infra; managed (Synadia Cloud) | Good (single binary, K8s-friendly, Helm charts) | Good (accounts, JetStream resource limits per account) |
| **ActiveMQ/Artemis** | Medium (Artemis significantly faster than Classic) | Low-Medium (Artemis: sub-ms to low-ms) | High (Artemis: journal-based, replication) | Queue-level | At-least-once; at-most-once; XA transactions | Medium (JVM tuning, journal configuration, address model) | Large (JMS ecosystem, many client protocols: AMQP, STOMP, MQTT, OpenWire) | Open source + infra | Moderate (K8s operators available, JVM resource overhead) | Moderate (addresses, security domains) |
| **ZeroMQ** | Very High (millions msg/s, zero-copy, no broker) | Very Low (microsecond-range, in-process) | None (no persistence, no broker) | Per-socket (in-order delivery per connection) | At-most-once (default); at-least-once (application-level) | Low (library, no infrastructure) but High (you build everything) | Medium (many language bindings, no connectors — it is a library) | Open source; zero infrastructure cost; high development cost | N/A (library, not infrastructure) | N/A (no broker) |
| **Redpanda** | Very High (Kafka-competitive, often better single-node) | Low (no JVM GC, lower tail latency than Kafka) | High (Raft-based replication) | Partition-level (Kafka-compatible) | At-least-once; exactly-once (Kafka-compatible) | Medium (single binary, no ZK/JVM, but still distributed system) | Large (Kafka API compatible, inherits Kafka ecosystem) | Open source (BSL → relicensed) + infra; managed (Redpanda Cloud) | Good (K8s operator, tiered storage) | Limited (same as Kafka — ACLs, quotas) |
| **Memphis** | Medium (limited by NATS JetStream backend) | Low (inherits NATS JetStream latency) | Medium-High (inherits JetStream durability) | Station-level | At-least-once | Low-Medium (GUI-driven, but project viability concerns) | Small (limited client SDKs, minimal connectors) | Open source + infra; was offering managed service | Good (K8s-native, Helm charts) | Limited |
| **Solace PubSub+** | Very High (hardware-accelerated appliances or software) | Very Low (sub-ms, deterministic) | High (guaranteed messaging with persistence) | Queue-level; topic-level configurable | At-least-once; at-most-once; JMS transactional | Medium (rich feature set, but well-documented; managed option available) | Large (JMS, MQTT, AMQP, REST, many enterprise integrations) | Commercial license or managed (Solace Cloud); enterprise pricing | Good (K8s operator, Docker, managed cloud) | Good (message VPNs, client profiles, ACLs) |
| **Chronicle Queue** | Extreme (millions msg/s, microsecond latency) | Ultra-Low (single-digit microsecond, no GC) | Medium (local disk, memory-mapped files, no replication) | Total (single writer, sequential) | At-most-once (single machine); replication is application-level | Low (library, no infrastructure) but specialised (Java/JVM only) | Small (Java only, Chronicle ecosystem) | Open source (community) + commercial (enterprise features) | N/A (library, designed for co-located processes) | N/A (library) |
| **Aeron** | Extreme (designed for low-latency, millions msg/s) | Ultra-Low (microsecond-range, zero-GC paths) | Configurable (Archive for persistence, Cluster for fault-tolerance) | Per-stream, per-session | At-most-once (UDP); reliable (Aeron protocol); exactly-once (Cluster) | High (complex configuration, deep networking knowledge needed) | Small (Java primary, C++ and .NET drivers, specialised community) | Open source (Apache 2.0) + commercial (Aeron Premium) | Limited (designed for bare metal / dedicated infra, not cloud-native) | N/A (transport-level, not a multi-tenant system) |

### How to Read This Table

A few notes before you screenshot this and present it in a meeting as if it were gospel:

1. **"Very High" throughput is not the same Very High across brokers.** Kafka's "Very High" and Aeron's "Extreme" exist on different scales. Kafka moves millions of messages per second across a distributed cluster to durable storage. Aeron moves millions of messages per second across a network with microsecond latency but without the same durability guarantees. Comparing them directly is like comparing a cargo ship and a speedboat — they both move fast, but "fast" means something different for each.

2. **Latency numbers depend on your definition of "latency."** Does the clock start when the producer calls `send()`? When the message hits the broker? When the consumer receives it? When the consumer acknowledges it? These are different measurements, and vendors are not always clear about which one they are quoting.

3. **Ops Complexity is the most subjective column.** A team that has run Kafka for five years will rate Kafka's operational complexity as "Medium." A team deploying it for the first time will rate it as "Please make it stop." Context matters more than the rating.

4. **Cost Model does not tell you what it will actually cost.** A per-message pricing model can be cheaper or more expensive than provisioned infrastructure, depending entirely on your traffic patterns. Do the maths for *your* workload, not the example on the pricing page.

---

## Notes and Caveats by Dimension

### Throughput Caveats

Throughput benchmarks are the most abused numbers in the messaging world. Every caveat from Chapter 10 applies, but the ones that most frequently invalidate comparisons are:

- **Message size.** A broker that handles 2 million 100-byte messages per second may struggle with 50,000 10KB messages per second. Always benchmark with representative message sizes.
- **Replication factor.** Virtually all vendor benchmarks quote throughput with minimal or no replication. Production systems run RF=3. The throughput difference can be 2-3x.
- **Durability settings.** Kafka with `acks=0` versus `acks=all` is a different broker in terms of throughput. Redis Streams with `WAIT` versus without `WAIT` is a different broker.
- **Consumer count.** Throughput is often measured at the producer. Adding consumers — especially with acknowledgments, processing logic, and back-pressure — changes the picture.

### Latency Caveats

- **JVM warm-up.** JVM-based brokers (Kafka, Pulsar, ActiveMQ) have a warm-up period where JIT compilation improves performance. Benchmarks that include cold-start latency look worse than steady-state.
- **Coordinated omission.** If a benchmark tool waits for the previous request to complete before sending the next one, it understates tail latency. Any benchmark that does not address coordinated omission should be viewed with suspicion.
- **Batching.** Many brokers batch messages for efficiency. Batching improves throughput at the cost of latency. A broker's "low latency" mode may have dramatically different throughput than its "high throughput" mode.

### Durability Caveats

- **Default configurations lie.** Kafka's default `acks=1` is not durable. RabbitMQ's classic queues without publisher confirms can lose messages on crash. Always check what "durable" means in the broker's default configuration versus its recommended production configuration.
- **Replication is not backup.** Replication protects against node failure. It does not protect against bugs that corrupt data on all replicas simultaneously, or against an operator who accidentally deletes a topic.

### Ordering Caveats

- **Ordering and retries are enemies.** If message A fails and is retried, it arrives after message B. Your "ordered" stream is now unordered. Kafka's `max.in.flight.requests.per.connection=1` prevents this but reduces throughput. Other brokers have analogous trade-offs.
- **Consumer parallelism destroys ordering.** Even if the broker delivers messages in order, a consumer that processes them in a thread pool has destroyed that ordering at the application level.

### Delivery Semantics Caveats

- **Exactly-once has a scope.** Kafka's exactly-once works within Kafka (topic to topic via Streams). The moment you write to an external system, you need application-level deduplication. This is true for every broker.
- **At-least-once is the pragmatic default.** With idempotent consumers, at-least-once is cheaper and more broadly applicable than exactly-once. Design for idempotency first, then evaluate whether you need exactly-once semantics.

---

## What the Table Does Not Tell You

The comparison matrix captures the measurable, the quantifiable, the things you can put in a spreadsheet. It does not capture the soft factors that, in practice, often matter more than raw performance numbers. These are the dimensions that do not fit in a cell.

### Documentation Quality

There is a spectrum from "comprehensive, well-organised, with practical examples and troubleshooting guides" to "auto-generated API docs with no context and a README that was last updated before the current major version." Where a broker falls on this spectrum determines how quickly your team becomes productive and how efficiently they debug production issues.

Kafka's documentation is extensive but can feel like reference material rather than guidance — you need to already know what you are looking for. RabbitMQ's documentation is genuinely good: well-structured, honest about trade-offs, and written by people who understand that operators need different information than developers. NATS's documentation is clean and focused. Pulsar's documentation has improved but still has gaps, especially for operational topics. The cloud providers (AWS, GCP, Azure) have documentation that is thorough but spread across dozens of service pages and can be hard to navigate. Solace's documentation is enterprise-grade — comprehensive, if somewhat formal.

### Community Vibe

Communities have cultures, and those cultures affect your experience when you need help.

Kafka's community is large but fragmented between the Apache project, Confluent's ecosystem, and various third-party tools. Help is available, but you may need to search in several places. RabbitMQ's community is friendly and helpful, with a long tradition of answering questions on mailing lists and forums. NATS's community is small but enthusiastic, with the core team being unusually responsive on GitHub and Slack. Pulsar's community is growing and technically strong but smaller than Kafka's. The cloud-native services have "communities" in the form of AWS re:Post and Google Groups, which is to say, professional support forums rather than communities in the social sense.

### Hiring Pool

If you need to hire someone who knows your broker, the size of the candidate pool matters.

Kafka expertise is the most available — it is on countless CVs, and "I know Kafka" has become a standard line item for backend engineers (though the depth of knowledge varies from "I used a Kafka consumer in a Spring Boot project" to "I can tune ISR replication and debug consumer lag at the partition level"). RabbitMQ expertise is also widely available. Redis expertise is everywhere, though Redis Streams-specific expertise is rarer. NATS, Pulsar, Redpanda, and most other brokers have smaller talent pools. For niche systems (Chronicle Queue, Aeron, Solace), you are hiring for general distributed systems expertise and training on the specific tool.

### The "2 AM Factor"

When your messaging system is broken at 2 AM, what is your experience going to be like?

This is a function of error messages (are they helpful?), observability (can you see what is wrong?), debugging tools (can you inspect the state?), and recovery procedures (can you fix it without losing data?). Some brokers fail gracefully with clear error messages and well-documented recovery procedures. Others fail with cryptic stack traces and recovery procedures that amount to "restart everything and hope."

Kafka's failure modes are well-documented because thousands of organisations have experienced them and written about them. RabbitMQ's management UI lets you see queue state and connection status, which is invaluable during incidents. NATS provides clear server logs and a monitoring endpoint. Cloud-managed services outsource the 2 AM problem to the cloud provider, which is worth its weight in gold for small teams.

### Momentum and Trajectory

Is the project gaining contributors, features, and adoption? Or is it in maintenance mode, slowly declining, or at risk of being abandoned?

As of this writing: Kafka is mature and evolving (KRaft replacing ZooKeeper, tiered storage maturing). RabbitMQ is stable under Broadcom's stewardship, though the acquisition has created some community uncertainty. Pulsar is growing but faces the challenge of operational complexity. NATS is on an upward trajectory with JetStream gaining adoption. Redpanda is actively developing and competing aggressively with Kafka. The cloud-native services (SNS/SQS, Pub/Sub, Event Hubs) are evolving steadily as part of their respective cloud platforms. Memphis's trajectory is uncertain. Solace continues to serve its enterprise niche.

---

## Protocol Support Comparison

Which wire protocols does each broker speak? This matters when you have existing clients, when you need to integrate with third-party systems, or when you want to avoid client library lock-in.

| Broker | AMQP 0.9.1 | AMQP 1.0 | MQTT | Kafka Protocol | HTTP/REST | gRPC | STOMP | Custom/Other |
|---|---|---|---|---|---|---|---|---|
| **Kafka** | — | — | — | Native | Confluent REST Proxy | — | — | Kafka binary protocol |
| **RabbitMQ** | Native | Plugin | Plugin | — | Management API | — | Plugin | — |
| **Pulsar** | — | — | Plugin (Pulsar-MQTT) | KoP (Kafka on Pulsar) | REST Admin API | — | — | Pulsar binary protocol |
| **SNS/SQS** | — | — | — | — | Native (AWS API) | — | — | AWS SDK protocol |
| **EventBridge** | — | — | — | — | Native (AWS API) | — | — | AWS SDK protocol |
| **Google Pub/Sub** | — | — | — | — | Native (REST) | Native (gRPC) | — | — |
| **Azure Event Hubs** | Yes (native) | Yes (native) | — | Yes (compatibility layer) | REST | — | — | — |
| **Redis Streams** | — | — | — | — | — | — | — | RESP (Redis protocol) |
| **NATS/JetStream** | — | — | Via gateway | — | Via WebSocket | — | — | NATS text protocol |
| **ActiveMQ/Artemis** | — | Artemis native | Artemis plugin | — | Jolokia REST | — | Both support | OpenWire (Classic) |
| **ZeroMQ** | — | — | — | — | — | — | — | ZMTP (ZeroMQ protocol) |
| **Redpanda** | — | — | — | Native (compatible) | HTTP Proxy (Pandaproxy) | — | — | Kafka binary protocol |
| **Solace PubSub+** | — | Yes | Yes (native) | — | Yes (native) | — | — | SMF (Solace Message Format) |
| **Aeron** | — | — | — | — | — | — | — | Aeron protocol (SBE) |

### Reading the Protocol Table

A few patterns emerge:

- **Kafka's protocol is becoming a de facto standard.** Redpanda implements it natively. Pulsar has KoP. Azure Event Hubs has a compatibility layer. When a broker adds "Kafka compatibility," it is an acknowledgment of Kafka's market position — your existing Kafka clients become a migration path.
- **AMQP has two versions and they are not the same thing.** AMQP 0.9.1 (RabbitMQ's native protocol) and AMQP 1.0 (an OASIS standard, ActiveMQ Artemis's native protocol) are different protocols with different semantics. "Supports AMQP" without a version number is ambiguous and possibly deceptive.
- **Multi-protocol brokers offer flexibility at the cost of cognitive load.** Solace, ActiveMQ Artemis, and RabbitMQ (with plugins) can speak multiple protocols. This is valuable for integration scenarios but means the operational team needs to understand the semantics and limitations of each protocol on that broker, not just the protocol in general.
- **HTTP/REST support is the great equaliser.** Almost every language and platform can make HTTP requests. Brokers with HTTP interfaces are accessible from serverless functions, legacy systems, and environments where installing a native client library is impractical. The trade-off is performance — HTTP is not the most efficient transport for high-throughput messaging.

---

## Language and Client Library Support

Having a broker is useless if you cannot talk to it from your programming language. Official, maintained client libraries matter — a community client with three GitHub stars and a last commit from 2021 is a liability, not a feature.

| Broker | Java/JVM | Python | Go | .NET/C# | JavaScript/Node.js | Rust | C/C++ | Ruby | PHP |
|---|---|---|---|---|---|---|---|---|---|
| **Kafka** | Official (excellent) | confluent-kafka-python (librdkafka) | confluent-kafka-go (librdkafka) | confluent-kafka-dotnet (librdkafka) | kafkajs / confluent-kafka-js | rdkafka (community, good) | librdkafka (reference) | ruby-kafka (community) | php-rdkafka (community) |
| **RabbitMQ** | Official (amqp-client) | pika (maintained) | amqp091-go (official) | RabbitMQ.Client (official) | amqplib (community, excellent) | lapin (community, good) | rabbitmq-c (official) | bunny (community, excellent) | php-amqplib (community) |
| **Pulsar** | Official (excellent) | Official | Official | Official (DotPulsar) | Official (Node.js) | Community | Official (C++) | Community | Community |
| **SNS/SQS** | AWS SDK | boto3 | AWS SDK | AWS SDK | AWS SDK | AWS SDK | AWS SDK | AWS SDK | AWS SDK |
| **Google Pub/Sub** | Official | Official | Official | Official | Official | Community | Community (C++) | Official | Official |
| **Azure Event Hubs** | Official | Official | Official (azeventhubs) | Official | Official | Community | Community | Community | Community |
| **Redis Streams** | Jedis, Lettuce | redis-py | go-redis | StackExchange.Redis | ioredis | redis-rs | hiredis | redis-rb | phpredis |
| **NATS/JetStream** | Official (jnats) | Official (nats.py) | Official (nats.go) | Official (nats.net) | Official (nats.js) | Official (nats.rs) | Official (nats.c) | Official (nats-pure) | Community |
| **ActiveMQ/Artemis** | Official (JMS) | stomp.py / proton | proton (AMQP 1.0) | NMS / AMQP.Net Lite | stompit / rhea | Community | proton-c (Qpid) | stomp (gem) | stomp-php |
| **ZeroMQ** | JeroMQ (pure Java) | pyzmq (official) | go-zeromq / pebbe/zmq4 | NetMQ (community, excellent) | zeromq.js | rust-zmq | czmq / libzmq (reference) | ffi-rzmq | php-zmq |
| **Redpanda** | Kafka clients (compatible) | Kafka clients (compatible) | Kafka clients (compatible) | Kafka clients (compatible) | Kafka clients (compatible) | Kafka clients (compatible) | Kafka clients (compatible) | Kafka clients (compatible) | Kafka clients (compatible) |
| **Solace PubSub+** | Official (JCSMP, JMS) | Official | Official | Official | Official | Community | Official (CCSMP) | Community | Community |
| **Chronicle Queue** | Official (Java only) | — | — | — | — | — | — | — | — |
| **Aeron** | Official (primary) | — | — | Community | — | Community | Official (C) | — | — |

### Reading the Client Library Table

- **Kafka's client ecosystem is unmatched in breadth**, largely thanks to librdkafka — a C library that provides the foundation for clients in Python, Go, .NET, and others. Redpanda inherits this ecosystem entirely because it speaks the Kafka protocol.
- **Cloud provider SDKs are comprehensive but vendor-locked.** AWS, Google, and Azure provide official SDKs for every major language. The quality is consistently high, but your code is coupled to that cloud provider's API.
- **NATS has invested heavily in official clients.** The NATS team maintains official clients for eight languages, which is unusual for a project of its size and reflects a deliberate strategy to reduce the friction of adoption.
- **ZeroMQ's client story reflects its nature as a library.** The reference implementation is in C (libzmq), and most language bindings are FFI wrappers around it. The quality is generally good, but debugging issues sometimes requires understanding the C layer beneath your language's abstraction.
- **Chronicle Queue and Aeron are JVM-first by design.** If you are not on the JVM, these are not practical options. This is not a limitation — it is a deliberate design choice. When you are optimising for microsecond latency and zero-GC, you are writing Java (or C++), full stop.
- **"Community" does not mean "bad."** Some community clients are excellent (bunny for RabbitMQ in Ruby, NetMQ for ZeroMQ in .NET). But community clients carry inherent risk: they may not keep up with broker protocol changes, and they may be maintained by one person whose interests could shift. Check the commit history and issue response time before depending on a community client in production.

---

## The Meta-Comparison: What Kind of Tool Are You Looking At?

Before comparing individual features, it helps to understand that these tools fall into fundamentally different categories. Comparing Kafka to ZeroMQ is like comparing PostgreSQL to SQLite — they are both "databases" in the way that both a container ship and a kayak are both "boats."

| Category | Brokers | What They Are | When to Compare Them |
|---|---|---|---|
| **Distributed log / Event streaming** | Kafka, Redpanda, Pulsar | Persistent, partitioned, high-throughput event logs | When your primary need is durable, replayable event streaming |
| **Traditional message broker** | RabbitMQ, ActiveMQ/Artemis, Solace | Message routing, queuing, exchange patterns | When you need flexible routing, protocol diversity, or enterprise integration |
| **Cloud-managed messaging** | SNS/SQS, EventBridge, Google Pub/Sub, Azure Event Hubs | Fully managed, no infrastructure to operate | When operational simplicity trumps all and you are committed to a cloud provider |
| **Lightweight / Embeddable** | Redis Streams, NATS/JetStream, Memphis | Low ceremony, quick to deploy, smaller footprint | When you want messaging without the operational weight of Kafka or Pulsar |
| **Messaging library** | ZeroMQ, Chronicle Queue, Aeron | Libraries you embed in your application, not infrastructure you deploy | When you need extreme performance and are willing to build your own infrastructure semantics |

Comparing tools within the same category is meaningful. Comparing tools across categories is meaningful only if you are genuinely deciding between fundamentally different approaches, which is a design decision, not a feature comparison.

---

## Final Note on the Matrix

No comparison matrix, however detailed, can substitute for benchmarking with your own workload on your own infrastructure with your own team. The matrix tells you where to look. It does not tell you what you will find when you get there.

The brokers that score "High" on every dimension do not exist. Every tool in this table made trade-offs, and those trade-offs are *features*, not *bugs* — they are the reason the tool is good at what it is good at. Kafka is operationally complex *because* it provides the throughput, durability, and ecosystem richness that require that complexity. ZeroMQ has no persistence *because* persistence would add latency that contradicts its design goals. SNS/SQS costs money *because* someone else is doing the operational work you are not doing.

The best broker for you is the one whose trade-offs align with your priorities. The matrix helps you see the trade-offs. The decision is yours.
