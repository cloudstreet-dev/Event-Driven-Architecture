# Evaluation Framework

Every message broker's marketing page says the same thing: fast, reliable, scalable, easy to operate. This is approximately as useful as a restaurant describing its food as "delicious." You need a framework — a structured set of criteria that lets you compare brokers on dimensions that actually matter for *your* workload, *your* team, and *your* budget. Otherwise you are choosing infrastructure based on blog post popularity and conference talk charisma, which is how organisations end up running Apache Kafka for a system that processes twelve messages per hour.

This chapter defines the evaluation framework we will use for every broker in Part 2. We are not going to rank brokers on a single axis, because single-axis rankings are how you end up with headlines like "Kafka vs RabbitMQ: Which Is Better?" — a question roughly as answerable as "Hammer vs Screwdriver: Which Is Better?" The answer, as always, is: it depends. But it depends on *specific, measurable things*, and that is what this chapter is about.

---

## Why a Framework Matters

The problem with choosing a message broker is not a lack of options. It is a surplus of options combined with a deficit of honest, apples-to-apples comparison. Every broker occupies a slightly different point in the design space. Some optimise for throughput. Some optimise for routing flexibility. Some optimise for operational simplicity. Some optimise for the VC pitch deck and will figure out the rest later.

Without a framework, broker evaluation degenerates into one of the following failure modes:

1. **The "My Last Job" heuristic.** You used Kafka at your previous company. It worked. You use Kafka again. This is fine until it is not — your new workload may have completely different characteristics.

2. **The benchmarketing trap.** You read a vendor benchmark showing 2 million messages per second. You did not notice it was running on 96 cores with messages the size of a TCP ACK, no replication, no durability, and consumers that discard every message immediately. In production you will get a tenth of that. Maybe.

3. **The "what does Google use?" fallacy.** Google uses a custom-built system you cannot buy. But even if you could, you are not Google. You do not have Google's traffic, Google's budget, or Google's army of SREs. Stop optimising for problems you do not have.

4. **The feature checkbox.** The broker supports exactly-once delivery. Great. Except "supports" means "there is a configuration option that, when combined with idempotent consumers, transactional producers, and a very specific set of operational practices, approximates exactly-once semantics under conditions that rarely hold in practice." The checkbox does not capture this nuance.

A framework forces you to ask the right questions, weight them according to your actual needs, and make trade-offs explicitly rather than accidentally.

---

## Dimension 1: Throughput

Throughput is the most frequently cited and most frequently misunderstood broker metric. It comes in two flavours:

**Messages per second** — how many discrete messages the broker can handle. This matters when your messages are small and your bottleneck is per-message overhead (serialisation, routing decisions, acknowledgment processing).

**Bytes per second** — how much raw data the broker can move. This matters when your messages are large (images, documents, fat JSON blobs) and your bottleneck is I/O bandwidth.

The distinction is critical. A broker that excels at 1KB messages may choke on 1MB messages, and vice versa. Always benchmark with messages that resemble your actual workload.

### Burst vs Sustained

Most systems do not produce traffic at a constant rate. They have peaks — Black Friday, market open, batch job completion, the moment a popular notification goes out. You need to know two things:

- **Sustained throughput**: what the broker can handle indefinitely without degradation.
- **Burst throughput**: what it can absorb for short periods before backpressure kicks in, latency spikes, or things start falling over.

A broker with excellent sustained throughput but no burst capacity will punish you during traffic spikes. A broker with excellent burst capacity but mediocre sustained throughput is living on borrowed time during prolonged load.

### What to Watch For

- **Replication cost.** Most benchmarks quote throughput with replication factor 1 (i.e., no replication). In production, you will run replication factor 3. This typically cuts throughput significantly — how much depends on the broker's replication protocol.
- **Acknowledgment mode.** "Fire and forget" throughput is always higher than "wait for durable acknowledgment" throughput. Make sure the benchmark matches your durability requirements.
- **Consumer throughput vs producer throughput.** They are not always symmetric. Some brokers are write-optimised; some are read-optimised.
- **Partition/queue count.** Throughput often scales with the number of partitions or queues, but at some point the overhead of managing many partitions exceeds the parallelism benefit.

---

## Dimension 2: Latency

If throughput is about how much, latency is about how fast. And the number that matters is almost never the average.

### Percentile Latency

- **p50 (median):** Half your messages are faster than this, half are slower. This is what most people think of as "latency," and it is the least interesting number.
- **p95:** 1 in 20 messages is slower than this. This is where user-visible pain begins.
- **p99:** 1 in 100 messages is slower than this. This is where SLAs live and die.
- **p99.9 and beyond (tail latency):** The worst-case scenario, excluding extreme outliers. Tail latency is caused by garbage collection pauses, disk flushes, rebalancing, leader elections, and other events that are rare individually but nearly guaranteed to happen at scale. If you process a million messages per day, your p99.9 latency is what *ten thousand* of those messages experience.

### Why Tail Latency Matters

In a microservices architecture, a single user request often fans out to multiple services. If each service adds a little tail latency, the overall request latency is dominated by the slowest component. This is the "tail at scale" problem. A broker with excellent p50 latency but terrible p99.9 will poison your entire request path.

### What to Watch For

- **GC pauses.** JVM-based brokers (Kafka, Pulsar) are susceptible to garbage collection pauses that show up as latency spikes. Tuning helps. Eliminating does not.
- **Batching trade-offs.** Many brokers batch messages for throughput efficiency. Batching improves throughput at the cost of latency — your message waits in a buffer until the batch fills or a timeout expires.
- **Network round trips.** How many round trips does it take to publish a message and receive an acknowledgment? The answer varies dramatically between brokers and protocols.
- **Coordinated omission.** A common benchmarking error where the measurement tool slows down during broker slowdowns, making latency look better than it actually is. If a vendor's latency benchmark does not mention coordinated omission, be sceptical.

---

## Dimension 3: Durability

Durability is the broker's answer to the question: "If something goes wrong, will my messages survive?"

### Failure Scenarios

"Something goes wrong" comes in degrees:

1. **Process crash.** The broker process dies and restarts. Messages in memory may be lost unless they were flushed to disk.
2. **Disk failure.** The physical storage device fails. Messages on that disk are gone unless replicated elsewhere.
3. **Node failure.** An entire machine goes down — power loss, hardware failure, kernel panic. Same as disk failure, but also affects any in-flight state.
4. **Network partition.** Nodes are alive but cannot communicate. The broker must decide whether to remain available (accepting writes that may diverge) or consistent (refusing writes until the partition heals). This is the CAP theorem in action, and every broker makes a different choice — or, more accurately, gives you a configuration knob to make the choice yourself.
5. **Datacenter failure.** An entire availability zone or region goes dark. Your messages survive only if they were replicated to another datacenter.

### What to Watch For

- **Default configuration.** Many brokers ship with durability settings optimised for performance, not safety. Kafka's default `acks=1` means the producer gets an acknowledgment after one broker writes to its page cache — not to disk. If that broker crashes before flushing, the message is gone. You want `acks=all` in most production scenarios, but you have to know to set it.
- **Replication protocol.** Synchronous replication is safer but slower. Asynchronous replication is faster but allows data loss during failover. Understand which one your broker uses and under what conditions.
- **fsync behaviour.** Does the broker actually call `fsync`, or does it rely on the OS page cache? The answer has enormous implications for durability after power loss.
- **Fencing and split-brain.** When a leader fails and a follower is promoted, can the old leader come back and accept writes that conflict with the new leader? Proper fencing prevents this; not all brokers implement it correctly.

---

## Dimension 4: Ordering Guarantees

"Messages arrive in order" is a statement that sounds simple and is anything but. Order is always relative to *something*:

**No ordering guarantee.** Messages may arrive in any order. This is the simplest model and the easiest to scale, but your consumers must be idempotent and order-independent.

**Partition-level ordering.** Messages within a single partition (or queue) are ordered. Messages across partitions are not. This is the Kafka model. If you need ordering for a specific entity (e.g., all events for order #1234), you route all events for that entity to the same partition using a partition key.

**Topic-level ordering.** All messages within a topic are totally ordered. This is simpler to reason about but limits throughput to what a single writer can produce, since total ordering requires a single serialisation point.

**Global ordering.** All messages across all topics are totally ordered. This is extremely expensive and almost never offered by distributed brokers. If you think you need global ordering, you probably need to reconsider your design.

### What to Watch For

- **Ordering and parallelism are in tension.** Stronger ordering guarantees mean fewer opportunities for parallel processing. A single partition gives you perfect ordering and zero parallelism. Pick your poison.
- **Redelivery breaks ordering.** If a message fails processing and is redelivered, it will arrive *after* messages that were originally behind it. Your "ordered" stream is now out of order. This is a fundamental tension in any system with retries.
- **Consumer concurrency.** Even if the broker delivers messages in order, if your consumer processes them concurrently (multiple threads pulling from the same partition), you have destroyed ordering at the application level.

---

## Dimension 5: Delivery Semantics

The holy trinity of messaging delivery guarantees:

**At-most-once.** The message is delivered zero or one times. Simple: fire and forget. If anything goes wrong, the message is lost. Appropriate for metrics, telemetry, and anything where losing a fraction of messages is acceptable.

**At-least-once.** The message is delivered one or more times. The broker retries until the consumer acknowledges receipt. This means duplicates are possible — your consumer must be idempotent, meaning processing the same message twice produces the same result as processing it once. This is the most common production setting and the one you should default to unless you have a specific reason not to.

**Exactly-once.** The message is delivered exactly one time. This is the white whale of distributed messaging. True exactly-once delivery in a distributed system is, by the laws of physics and distributed computing theory, impossible in the general case. What brokers actually offer is *exactly-once semantics* — the system behaves *as if* each message was delivered exactly once, through a combination of idempotent producers, transactional writes, and consumer offset management. It works. It is also slower, more complex, and more fragile than at-least-once with idempotent consumers. Use it when you genuinely need it (financial transactions, inventory updates). Do not use it because it sounds nice.

### What to Watch For

- **The scope of "exactly-once."** Kafka's exactly-once semantics apply within the Kafka ecosystem — from Kafka topic to Kafka topic via Kafka Streams. The moment you write to an external database, you are back to at-least-once unless you implement your own deduplication. No broker can guarantee exactly-once delivery to an arbitrary external system.
- **The cost of exactly-once.** Transactional producers and consumers add overhead — additional round trips, coordinator involvement, and reduced throughput. Benchmark with transactions enabled, not disabled.
- **Idempotency is your actual safety net.** Regardless of what the broker promises, design your consumers to be idempotent. It costs you almost nothing in most cases and protects you from an entire category of bugs.

---

## Dimension 6: Operational Complexity

This is the dimension that separates conference talks from production incidents. Every broker is easy to run in a Docker Compose file on your laptop. The question is what happens when you run it in production with real data, real traffic, and real failure modes.

### Deployment

- How many components do you need to deploy? Kafka historically required ZooKeeper — a separate distributed system with its own failure modes. Pulsar requires ZooKeeper *and* BookKeeper. RabbitMQ is a single binary with an Erlang runtime.
- Can you run it on Kubernetes? Is there a mature operator? Or does the broker have stateful requirements that fight Kubernetes's abstractions?
- What is the minimum viable cluster for production? Three nodes? Five? One node with a prayer?

### Monitoring

- What are the key metrics? Every broker has its own set of critical indicators — consumer lag, under-replicated partitions, queue depth, memory alarms.
- Is there a built-in dashboard, or do you need to configure Prometheus/Grafana/Datadog from scratch?
- How easy is it to correlate broker metrics with application-level symptoms?

### Upgrades

- Can you do rolling upgrades with zero downtime? This seems like a basic requirement, but the reality varies. Some brokers handle it gracefully. Others require careful partition leadership migration, client compatibility checks, and possibly ritual sacrifice.
- How frequently are new versions released? Is the upgrade path well-documented?
- Are there protocol version negotiations between clients and brokers?

### Staffing

This is the one nobody talks about in the evaluation spreadsheet. If your broker requires specialised expertise to operate, you need to hire or train people with that expertise. Kafka operations is a genuine specialisation. Erlang debugging is a genuine specialisation. BookKeeper tuning is a genuine specialisation. If you are a team of five and nobody has touched your broker's underlying technology, factor in the learning curve — or the consulting bill.

---

## Dimension 7: Ecosystem

A message broker does not exist in isolation. It lives in an ecosystem of:

- **Client libraries.** Are there official clients for your language? Are they well-maintained? Are there community clients, and if so, are they production-quality or weekend projects?
- **Connectors.** Can you stream data to and from your databases, data lakes, search indices, and cloud services without writing custom code? Kafka Connect has hundreds of connectors. Other brokers have fewer.
- **Stream processing.** Can you do lightweight transformations on the broker itself, or do you need a separate processing framework? Kafka has Kafka Streams and ksqlDB. Pulsar has Pulsar Functions. RabbitMQ has... a plugin for that, probably.
- **Schema management.** Is there a schema registry for enforcing contracts between producers and consumers?
- **Tooling.** Command-line tools, admin UIs, debugging utilities, performance testing tools.
- **Integration with observability stacks.** OpenTelemetry support, distributed tracing propagation, structured logging.

A broker with a rich ecosystem lets you build faster and integrate more easily. A broker with a thin ecosystem means you are writing more glue code and building more tooling yourself.

---

## Dimension 8: Cost

Cost has three layers, and most evaluations only look at the first one.

### Layer 1: Licensing and Infrastructure

- **Licensing.** Is the broker open source? Truly open source (Apache 2.0, MIT) or source-available with restrictions (BSL, SSPL)? Does the vendor offer a commercial edition with features you actually need?
- **Infrastructure.** How much compute, memory, and storage does the broker require? A broker that requires SSDs for acceptable performance costs more than one that is happy on spinning disks. A broker that requires 32GB of heap per node costs more than one that runs in 2GB.
- **Managed service pricing.** If you use a managed offering, what is the pricing model? Per message? Per byte? Per partition? Per hour? Managed services shift cost from ops headcount to cloud bills, but the total cost may be higher or lower depending on your usage patterns.

### Layer 2: Operational Cost

- **Staffing.** How many people does it take to keep this thing running? A complex broker that requires a dedicated team is more expensive than a simple one that your existing platform team can manage alongside other services.
- **Incident cost.** When things go wrong — and they will — how expensive is the outage? A broker that is hard to debug, hard to recover, or hard to failover extends your MTTR and increases the cost of every incident.

### Layer 3: Opportunity Cost

- **Time to market.** A broker with a steep learning curve delays your project. A broker with a rich ecosystem accelerates it.
- **Lock-in.** How much effort does it take to switch brokers later? If you have built your entire architecture around Kafka Streams and Schema Registry, migrating to Pulsar is not a weekend project. It is a quarter. Maybe two.

---

## Dimension 9: Community and Longevity

Will this broker exist in five years? This is not a trivial question.

- **Apache Foundation projects** (Kafka, Pulsar, ActiveMQ) have the backing of a foundation and a contributor community that outlives any single company. But foundation governance can also mean slow progress and design-by-committee.
- **Corporate-backed projects** (RabbitMQ under Broadcom, Redis under Redis Ltd) depend on the continued investment of their corporate steward. Corporate priorities shift. Acquisitions happen. Licence changes happen.
- **VC-funded startups** (various newer brokers) exist as long as the funding lasts and the business model works. Some will thrive. Some will pivot. Some will acqui-hire their engineering team and shut down the product.

Look at the contributor graph, the release cadence, the mailing list or forum activity, and the job market. If nobody is hiring for your broker, that is a signal — either it is so simple that nobody needs specialists (good) or nobody is using it (bad).

---

## Dimension 10: Cloud-Native Readiness

Like it or not, most new deployments target cloud environments, and increasingly Kubernetes.

- **Managed offerings.** Does a major cloud provider or the vendor offer a fully managed service? Managed services trade control for convenience, and for many teams, the trade is worth it.
- **Kubernetes operators.** Is there a mature, actively maintained operator? Operators handle the stateful lifecycle management that makes running distributed systems on Kubernetes bearable.
- **Tiered storage / cloud-native storage.** Can the broker offload cold data to object storage (S3, GCS)? This dramatically reduces storage costs for high-retention workloads.
- **Elasticity.** Can you scale the broker up and down in response to load? Or is it sized for peak and wasting resources the rest of the time?

---

## Dimension 11: Multi-Tenancy

If multiple teams or applications share a broker cluster — and they will, because running dedicated clusters for every team is expensive — you need multi-tenancy support.

- **Namespace isolation.** Can you create logical namespaces with separate access controls, quotas, and policies? Pulsar has first-class multi-tenancy with tenants and namespaces. Kafka has ACLs and quotas but no formal namespace concept. RabbitMQ has virtual hosts.
- **Resource quotas.** Can you limit the throughput, storage, or connection count for a specific tenant? Without quotas, one noisy team can starve everyone else.
- **Topic/queue policies.** Can you set retention, replication, and other policies per-tenant rather than cluster-wide?
- **Observability per tenant.** Can you see metrics broken down by tenant, or is everything aggregated?

Weak multi-tenancy means you end up running multiple clusters anyway, at which point you are paying the operational cost of multi-tenancy without the resource efficiency benefit.

---

## The Framework at a Glance

Here is the complete evaluation framework in table form. In subsequent chapters, we will score each broker against these dimensions.

| Dimension | Key Questions | Why It Matters |
|---|---|---|
| **Throughput** | Messages/sec? Bytes/sec? Burst vs sustained? | Can it handle your volume? |
| **Latency** | p50, p95, p99, p99.9? | Can it handle your speed requirements? |
| **Durability** | Replication? fsync? Datacenter failure? | Will you lose messages? |
| **Ordering** | None, partition, topic, global? | Can your consumers process correctly? |
| **Delivery Semantics** | At-most-once, at-least-once, exactly-once? | What happens when things fail? |
| **Operational Complexity** | Components, monitoring, upgrades, staffing? | What does it cost to keep running? |
| **Ecosystem** | Clients, connectors, tooling, stream processing? | What can you build without custom code? |
| **Cost** | Licensing, infrastructure, ops, opportunity? | What is the total cost of ownership? |
| **Community & Longevity** | Contributors, releases, governance, job market? | Will it outlive your project? |
| **Cloud-Native Readiness** | Managed services, K8s operators, tiered storage? | Does it fit your deployment model? |
| **Multi-Tenancy** | Namespaces, quotas, per-tenant policies? | Can teams share safely? |

---

## How to Use This Framework

The framework is not a scorecard where the highest total wins. Different workloads weight these dimensions differently:

- **High-throughput event streaming** (clickstream, IoT telemetry): weight throughput and cost heavily; latency and ordering may be less critical.
- **Financial transaction processing**: weight durability, ordering, and delivery semantics heavily; cost is secondary to correctness.
- **Microservice command/event bus**: weight operational complexity and ecosystem heavily; raw throughput is rarely the bottleneck.
- **Multi-team platform**: weight multi-tenancy and ecosystem heavily; you need a broker that scales organisationally, not just technically.

The chapters that follow will evaluate each broker honestly against this framework. Some brokers will shine in certain dimensions and stumble in others. That is not a failure of the broker — it is a reflection of the trade-offs inherent in distributed systems design. The goal is not to find the "best" broker. It is to find the best broker *for you*.

Let us begin.
