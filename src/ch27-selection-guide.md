# Selection Guide

You have read twenty-five chapters. You have absorbed throughput numbers, latency percentiles, durability guarantees, and enough operational caveats to fill a incident retrospective database. You can now recite the difference between AMQP 0.9.1 and AMQP 1.0 at dinner parties, which will not make you more popular but will make you more correct. The question remains: which broker should you actually use?

This chapter provides decision frameworks, not answers. Answers require context that a book cannot have — your team's skills, your budget, your existing infrastructure, your risk tolerance, your deadline, and whether you have a VP who once read a Confluent blog post and has Opinions. What this chapter *can* do is give you structured ways to narrow the field, practical heuristics for common scenarios, and enough honesty about the process to save you from the most common selection mistakes.

---

## The First Question: Do You Even Need a Message Broker?

Before evaluating brokers, evaluate whether you need one at all. This sounds obvious. It is not. A significant number of message broker deployments exist because someone said "we need to decouple our services" without examining whether the coupling was actually causing problems, or because an architect drew boxes and arrows on a whiteboard and one of the arrows was labelled "message queue" without anyone asking why.

You probably *do* need a message broker if:

- **Multiple consumers need the same event.** A new order is created, and the inventory service, billing service, analytics pipeline, and notification service all need to know. Without a broker, you are writing point-to-point integrations, and the next service that needs the event means modifying the producer. This is the strongest case for a broker.
- **You need to absorb traffic spikes.** Your producers generate work faster than your consumers can process it during peaks. A buffer between them prevents cascade failures and lets consumers process at their own pace.
- **You need durability for in-flight work.** If a consumer crashes, you need the work item to survive and be reprocessed. A database with a polling pattern can do this, but a broker does it with less latency and less polling overhead.
- **You need geographic distribution of events.** Events produced in one region need to be consumed in another. Brokers with replication and federation handle this; building it yourself is a distributed systems project you do not want.

You probably *do not* need a message broker if:

- **You have one producer and one consumer.** A direct HTTP call with a retry library, or a database table with a polling consumer, may be simpler and more appropriate. The overhead of deploying and operating a broker is not justified for a point-to-point integration.
- **Your "events" are actually request-response.** If the producer needs an immediate answer from the consumer, you do not have a messaging problem — you have an RPC problem. Use HTTP, gRPC, or a service mesh. Shoehorning request-response into a message broker adds latency and complexity for no benefit.
- **Your total message volume is tiny.** If you process a hundred messages a day, a PostgreSQL table with a `processed` boolean column and a cron job is a perfectly respectable architecture. It is boring, reliable, easy to debug, and requires zero additional infrastructure. Do not let anyone shame you into deploying Kafka for this.
- **You are a team of two and your deadline is next month.** A message broker is an additional system to learn, deploy, monitor, and debug. If you are resource-constrained and your architecture does not absolutely require asynchronous messaging, postpone the broker. You can always add one later. You cannot easily remove one that has become load-bearing.

The honest truth: most applications start without a broker and add one when the need becomes clear. The applications that start *with* a broker and never quite use it properly are more common than anyone admits.

---

## Decision Tree #1: By Primary Use Case

Start with what you are trying to do. Different use cases lead to fundamentally different parts of the broker landscape.

### Event Streaming and Log Processing

**The problem:** You need to capture, store, and process a high-volume stream of events — clickstream data, IoT telemetry, application logs, change data capture from databases, user activity tracking. Events need to be durable, replayable, and processable by multiple independent consumers.

**First choice:** Kafka or Redpanda. This is their home turf. The distributed log model — append-only, partitioned, consumer-driven — was designed for exactly this use case. Kafka has the larger ecosystem (Connect, Streams, Schema Registry, massive community). Redpanda offers Kafka API compatibility with a simpler operational profile (single binary, no JVM, no ZooKeeper).

**Cloud-native alternative:** Google Pub/Sub or Azure Event Hubs if you are committed to a cloud provider and want to eliminate operational burden. Event Hubs has Kafka wire compatibility, which makes migration feasible. Google Pub/Sub is excellent but uses its own API, so you are buying in fully.

**If scale is modest:** NATS JetStream. Lighter weight than Kafka, easier to operate, and capable enough for event streaming workloads that do not need the full Kafka ecosystem.

### Task Queuing and Background Jobs

**The problem:** You need to distribute units of work to a pool of workers. An order needs to be processed. An email needs to be sent. A report needs to be generated. Work items should be load-balanced across workers, retried on failure, and not lost if a worker crashes.

**First choice:** RabbitMQ. This is what it was built for. The AMQP model — messages routed through exchanges to queues, consumed by competing consumers, acknowledged on completion — maps directly to task queuing. Quorum queues provide durability. Dead-letter exchanges handle failed messages. The management UI lets you see queue depth and consumer state.

**Cloud-native alternative:** SQS. No infrastructure to operate, scales to infinity, integrates with Lambda for serverless processing. FIFO queues add ordering if you need it. The trade-off is vendor lock-in and slightly higher latency than a self-hosted broker.

**Serverless-specific:** QStash if your workers are serverless functions and you want HTTP push delivery with built-in retry. It is a specialised tool for a specialised environment.

**Minimal overhead:** Redis Streams with consumer groups. If you already have Redis in your stack, adding task queuing requires no additional infrastructure. The durability story is weaker than RabbitMQ's, so this is best for workloads where occasional message loss during a Redis failure is acceptable.

### Real-Time Messaging and Notifications

**The problem:** You need low-latency message delivery for real-time applications — chat, live updates, notifications, collaborative editing, gaming events. Messages should arrive quickly. Durability is secondary to speed.

**First choice:** NATS (Core, without JetStream). Sub-millisecond publish-subscribe with no persistence overhead. Wildcard subject routing gives you flexible topic hierarchies. The simplicity of the protocol and the performance of the implementation make it ideal for real-time scenarios.

**With persistence added:** NATS JetStream if you need some messages to be durable (offline notification delivery, message history) alongside the real-time flow.

**Enterprise / Multi-protocol:** Solace PubSub+. If you need real-time messaging with protocol diversity (MQTT for IoT clients, WebSocket for browsers, JMS for enterprise systems) and enterprise features (message VPNs, quality of service, guaranteed messaging).

**IoT-specific:** An MQTT broker (Mosquitto, EMQX, HiveMQ, or Mochi for embedded scenarios). MQTT was designed for constrained devices and unreliable networks. If your "real-time messaging" involves IoT devices, MQTT is the protocol and an MQTT broker is the natural choice.

### Financial and Ultra-Low-Latency

**The problem:** You are building trading systems, market data distribution, high-frequency pricing, or any system where microsecond latency matters and you are willing to invest significant engineering effort to achieve it.

**First choice:** Aeron for transport, Chronicle Queue for inter-process communication on the same machine. These are not general-purpose brokers — they are precision tools for teams that measure latency in microseconds and accept the engineering investment that entails. You are writing Java (or C++), you are tuning kernel parameters, and you are probably bypassing the network stack in creative ways.

**If milliseconds (not microseconds) are acceptable:** Solace PubSub+ with hardware appliances. Deterministic sub-millisecond latency without the bare-metal engineering effort of Aeron/Chronicle Queue. The trade-off is cost — Solace is enterprise-priced.

**If you think you need this category but are not in financial services:** You probably do not. Revisit your latency requirements. The engineering investment for microsecond-level messaging is enormous, and the tools in this category are designed for a very specific set of problems.

### Event Sourcing and CQRS

**The problem:** Your architecture stores state as a sequence of events rather than as current state. You need an event store, not a message broker — though you also need the ability to subscribe to events for building read models and triggering side effects.

**First choice:** EventStoreDB. It was literally built for this. Event streams, subscriptions, projections, and category streams are first-class concepts. If you are doing event sourcing, EventStoreDB is the most natural fit.

**Alternative:** Kafka as an event store. It works — Kafka's log model is conceptually similar to an event store. But you lose EventStoreDB's optimistic concurrency on individual streams, its projection engine, and its purpose-built tooling. You gain Kafka's ecosystem, throughput, and community.

**Library-level:** Eventuous (.NET) or Marten (.NET) for event sourcing without a dedicated event store, using PostgreSQL or EventStoreDB as the backing store.

---

## Decision Tree #2: By Constraint

Sometimes the use case is flexible but the constraints are not. Start with what you cannot change.

### "It Must Be Fully Managed — We Cannot Operate Infrastructure"

Your shortlist is: **SQS/SNS, EventBridge, Google Pub/Sub, Azure Event Hubs, Confluent Cloud, Redpanda Cloud, Solace Cloud**. Of these, the cloud-provider-native options (SQS, Pub/Sub, Event Hubs) have the deepest integration with their respective ecosystems. Confluent Cloud and Redpanda Cloud give you Kafka-compatible managed services without cloud provider lock-in (though you are locked into the vendor instead).

If you genuinely cannot operate messaging infrastructure — small team, no dedicated platform/SRE function, other priorities — this constraint alone eliminates most self-hosted options. Accept it and choose accordingly.

### "It Must Be Open Source — No Vendor Lock-In"

Your shortlist is: **Kafka, RabbitMQ, Pulsar, NATS, ActiveMQ/Artemis, ZeroMQ, Redis (with caveats about licensing changes)**. All are available under permissive or copyleft open-source licences. Note that "open source" and "free" are not the same thing — you are trading vendor lock-in for operational responsibility.

Be honest about *why* you need open source. If it is philosophical (you believe in open source), the constraint is real and non-negotiable. If it is practical (you want to avoid vendor pricing), factor in the cost of operating the infrastructure yourself. Self-hosted Kafka is "free" in the same way that a free puppy is free.

### "We Are Already Running X — What Fits?"

**Already running Kafka?** Stay on Kafka for new use cases that fit the streaming model. Add RabbitMQ or SQS if you need task queuing with flexible routing — do not force Kafka into a task queue pattern. If Kafka's operational burden is the pain point, evaluate Redpanda as a drop-in replacement.

**Already running RabbitMQ?** Stay on RabbitMQ for task queuing and simple pub/sub. Add Kafka or Redpanda if you need event streaming with replay, log compaction, or the Kafka Connect ecosystem.

**Already running Redis?** Redis Streams can handle basic messaging without adding infrastructure. If you outgrow Redis Streams' durability and feature set, the natural graduation path is to RabbitMQ (for task queuing) or Kafka (for event streaming).

**Already running on AWS?** SQS/SNS and EventBridge are there, they are managed, and they integrate with everything. Use them unless you have a specific need they cannot meet (Kafka-level throughput, replay from arbitrary offsets, cross-cloud portability).

### "Our Team Knows Language X"

**Java team:** Everything is available. Kafka's native client is Java. Pulsar, ActiveMQ, Solace, Chronicle Queue, and Aeron all have first-class Java support.

**Go team:** NATS (written in Go, official Go client is excellent), Kafka (via confluent-kafka-go or Sarama), RabbitMQ (official Go client). Watermill as an abstraction layer if you want to stay agnostic.

**.NET team:** RabbitMQ (excellent .NET client), Kafka (confluent-kafka-dotnet), NATS (official .NET client), Eventuous or Marten for event sourcing. Azure Event Hubs if you are in the Microsoft ecosystem.

**Python team:** Kafka (confluent-kafka-python), RabbitMQ (pika), SQS/SNS (boto3), Google Pub/Sub (official), Redis Streams (redis-py). Python's client ecosystem is broad enough that language is not usually the constraint.

**Node.js team:** KafkaJS (or confluent-kafka-javascript), amqplib (RabbitMQ), AWS SDK, NATS (official). The JavaScript client ecosystem is adequate for all major brokers.

---

## Decision Tree #3: By Scale

### Startup / Small Team (1-10 engineers)

**Priorities:** Simplicity, low operational overhead, fast time to value.

**Recommendations:**
- **Default choice:** A managed service. SQS if you are on AWS. Google Pub/Sub if you are on GCP. Do not operate messaging infrastructure if you can avoid it. Your engineering time is too valuable to spend on broker operations.
- **If you must self-host:** NATS or Redis Streams. Both are simple to deploy, easy to operate, and capable enough for startup-scale workloads. NATS as a single binary with JetStream enabled covers both pub/sub and persistent messaging.
- **Avoid:** Kafka, Pulsar, or any broker that requires more than one component to deploy. You do not have the team to operate it, and you do not have the traffic to justify it.

### Mid-Size Organisation (10-100 engineers, multiple teams)

**Priorities:** Multi-team usage, reasonable operational complexity, growing ecosystem needs.

**Recommendations:**
- **Event streaming:** Kafka (if you have or can hire Kafka expertise) or a managed Kafka-compatible service (Confluent Cloud, Redpanda Cloud, Amazon MSK). The ecosystem (Connect, Schema Registry) becomes valuable at this scale.
- **Task queuing:** RabbitMQ or SQS. RabbitMQ if you want self-hosted with full control. SQS if you want zero ops.
- **Platform play:** If multiple teams need messaging, invest in a platform. Set up shared infrastructure with multi-tenancy, monitoring, and self-service topic/queue creation. Pulsar's native multi-tenancy is appealing here if you can stomach the operational complexity.

### Enterprise (100+ engineers, strict compliance, multi-region)

**Priorities:** Reliability, compliance, multi-tenancy, vendor support, multi-region.

**Recommendations:**
- **Core streaming platform:** Kafka (with Confluent support) or Pulsar (with StreamNative support). Enterprise support contracts matter when production incidents have business-level consequences.
- **Multi-protocol needs:** Solace PubSub+ if you need to bridge JMS, MQTT, AMQP, and REST under a single platform with enterprise-grade support and features.
- **Cloud-native:** Azure Event Hubs (Kafka-compatible) or Google Pub/Sub with enterprise support. Cloud-managed services with SLAs and compliance certifications reduce audit friction.
- **Event sourcing:** EventStoreDB with commercial support (Event Store Cloud) for event-sourced domains.

### Hyperscale (Thousands of engineers, millions of messages per second, global distribution)

**Priorities:** Throughput, global distribution, operational maturity, custom tooling.

**Recommendations:**
- At this scale, you are probably running Kafka or a custom system. You have dedicated teams for messaging infrastructure. You have opinions about partition assignment strategies and consumer rebalancing protocols. You have already read this book and formed your own conclusions.
- The question at hyperscale is not which broker to choose but how to operate it at scale: automated partition rebalancing, tiered storage for cost management, cross-region replication with conflict resolution, and custom monitoring that alerts before users notice problems.
- Consider Redpanda if you want Kafka compatibility with simpler operations. Consider Pulsar if you need native multi-tenancy for a large number of teams. Consider building a platform team whose sole job is messaging infrastructure.

---

## "Use This When" Quick Reference

One-liner recommendations for when each broker is the right call. These are opinionated. They are also, in the author's experience, correct more often than not.

| Broker | Use This When... |
|---|---|
| **Kafka** | You need durable, high-throughput event streaming with a massive ecosystem and can invest in operations. |
| **RabbitMQ** | You need flexible message routing, task queuing, or a multi-protocol broker that is well-understood and well-documented. |
| **Pulsar** | You need Kafka-like streaming with native multi-tenancy and geo-replication, and you can handle the operational complexity. |
| **SQS/SNS** | You are on AWS and want managed messaging that scales to infinity with zero operational burden. |
| **EventBridge** | You need event-driven integration between AWS services, SaaS applications, or microservices with rule-based routing. |
| **Google Pub/Sub** | You are on GCP and want managed messaging with strong ordering support and exactly-once per-subscription semantics. |
| **Azure Event Hubs** | You are on Azure and want managed event streaming with Kafka wire compatibility. |
| **Redis Streams** | You already have Redis and need lightweight messaging without deploying additional infrastructure. |
| **NATS/JetStream** | You want a simple, fast, operationally lightweight messaging system that covers both ephemeral and persistent messaging. |
| **ActiveMQ/Artemis** | You need JMS compliance, XA transactions, or enterprise Java integration patterns. |
| **ZeroMQ** | You need the fastest possible inter-process messaging and are willing to build infrastructure semantics yourself. |
| **Redpanda** | You want Kafka compatibility without the JVM, ZooKeeper, or the operational complexity. |
| **Memphis** | You want a developer-friendly layer over NATS JetStream and accept the project viability risk. (Evaluate carefully.) |
| **Solace PubSub+** | You need enterprise messaging with multi-protocol support, guaranteed delivery, and commercial support. |
| **Chronicle Queue** | You need microsecond-latency inter-process messaging on a single machine (JVM only). |
| **Aeron** | You need microsecond-latency network messaging and are building a system where latency justifies engineering complexity. |
| **EventStoreDB** | You are doing event sourcing and want a database designed for it, not a general-purpose broker repurposed for it. |
| **RocketMQ** | You need transaction messages baked into the broker and are comfortable with a Java-centric, China-origin ecosystem. |

---

## Migration Paths: Common Broker-to-Broker Migrations

Migrations happen. Requirements change, teams grow, workloads evolve, and the broker that was perfect two years ago may not be perfect today. Here is an honest assessment of common migration paths.

### RabbitMQ to Kafka

**Why it happens:** The team outgrows RabbitMQ's throughput ceiling, needs event replay, or wants the Kafka ecosystem (Connect, Streams, Schema Registry) for data pipeline use cases.

**Difficulty: High.** The mental model is different. RabbitMQ is push-based with routing logic (exchanges, bindings). Kafka is pull-based with partitioned logs. Consumer patterns change fundamentally. Your RabbitMQ consumers that process and acknowledge individual messages become Kafka consumers that manage offsets and batches. Routing logic that lived in RabbitMQ's exchange topology now lives in your application or in Kafka Streams.

**Advice:** Migrate use case by use case, not all at once. Run both brokers in parallel. Start with new use cases on Kafka. Migrate existing use cases only when the RabbitMQ limitations are concretely painful, not theoretically concerning.

### Kafka to Redpanda

**Why it happens:** Kafka's operational complexity (ZooKeeper, JVM tuning, partition rebalancing) is a pain point, and the team wants Kafka compatibility with a simpler operational model.

**Difficulty: Low to Medium.** Redpanda is Kafka API-compatible, so clients require minimal or no changes. The migration is primarily an infrastructure swap: stand up Redpanda, migrate data (MirrorMaker or Redpanda's tools), switch producers and consumers, decommission Kafka. The risk is in the edge cases — Kafka compatibility is very good but not 100%. Test your specific client usage patterns thoroughly.

### SQS to RabbitMQ (or vice versa)

**Why it happens:** Multi-cloud migration, cost optimisation, or the need for features one has that the other lacks (RabbitMQ's exchange routing, SQS's unlimited scalability).

**Difficulty: Medium.** The concepts map reasonably well (SQS queue ≈ RabbitMQ queue, SNS topic ≈ RabbitMQ fanout exchange). The API is completely different, so all producer and consumer code changes. The operational model changes dramatically — from zero ops (SQS) to self-hosted (RabbitMQ), or vice versa.

### ActiveMQ to RabbitMQ or Kafka

**Why it happens:** ActiveMQ Classic is aging and has known performance and reliability limitations. Teams migrate to RabbitMQ for a modern traditional broker or to Kafka for event streaming.

**Difficulty: Medium to High.** If the codebase uses JMS extensively, migrating to Kafka means abandoning JMS (unless you use a JMS-to-Kafka bridge, which adds complexity). Migrating to Artemis is lower friction since Artemis supports JMS natively. Migrating to RabbitMQ means switching to AMQP, which is a protocol change but a manageable one.

### Any Broker to a Cloud-Managed Service

**Why it happens:** The team is tired of operating infrastructure and wants to hand the pager to AWS/GCP/Azure.

**Difficulty: Varies.** Kafka to Amazon MSK or Confluent Cloud is relatively smooth (same protocol, similar operational model). Kafka to SQS/SNS is a fundamental architecture change. RabbitMQ to Amazon MQ (which runs RabbitMQ) is straightforward. RabbitMQ to SQS requires rewriting consumer patterns.

**The meta-advice on migration:** Every migration takes longer and costs more than you expect. Every migration discovers assumptions that were baked into the old broker's behaviour and never documented. Budget twice the time you think you need, and keep the old broker running in parallel until you are *certain* the migration is complete. "Certain" means you have run in production for at least two full business cycles (including whatever your peak period is) without issues.

---

## The Multi-Broker Reality

Here is something the vendor marketing will never tell you: most organisations of any significant size end up running more than one messaging system. This is not a failure of architecture. It is a rational response to having multiple, genuinely different messaging needs.

A typical mid-to-large organisation might run:

- **Kafka** for the core event streaming platform — user activity, change data capture, metrics pipelines.
- **SQS** for background job queuing in AWS-deployed services — simple, managed, scales without thought.
- **Redis Pub/Sub or NATS** for real-time notifications between microservices — low-latency, ephemeral messages that do not need durability.
- **An MQTT broker** for IoT device communication — different protocol, different clients, different network assumptions.

Is this ideal? No. Each system requires its own monitoring, expertise, and operational procedures. But the alternative — forcing all messaging through a single broker — creates a different set of problems: you contort the broker to fit use cases it was not designed for, you create a single point of failure for your entire organisation, and you optimise for none of your use cases because you are trying to optimise for all of them.

The practical approach is to standardise on *as few brokers as possible* while accepting that "one broker for everything" is a platonic ideal, not a realistic goal. Define clear criteria for when a new messaging system is justified, and resist the urge to adopt a new broker for every new project. But also resist the urge to force Kafka into being a task queue or RabbitMQ into being an event store.

---

## Building Your Evaluation: A Practical Scoring Template

When it is time to make a decision for a specific project, use a structured evaluation rather than a gut feeling. Here is a template.

### Step 1: Weight the Dimensions

Not all dimensions matter equally for every project. Assign weights (1-5) based on your specific needs:

| Dimension | Weight (1-5) | Notes |
|---|---|---|
| Throughput | ___ | What volume do you actually need? Not aspirational, actual. |
| Latency (p99) | ___ | What latency is the user experience sensitive to? |
| Durability | ___ | What is the cost of losing a message? Negligible, annoying, or catastrophic? |
| Ordering | ___ | Does your processing logic require ordering? Or can consumers handle out-of-order? |
| Delivery Semantics | ___ | Do you need exactly-once, or can you design idempotent consumers? |
| Ops Complexity | ___ | How much operational capacity does your team have? |
| Ecosystem | ___ | Do you need connectors, stream processing, schema management? |
| Cost | ___ | What is your budget for infrastructure and staffing? |
| Cloud-Native | ___ | Are you deploying to Kubernetes? Do you need a managed service? |
| Multi-Tenancy | ___ | Will multiple teams share the broker? |

### Step 2: Score Each Candidate

For each candidate broker, score it 1-5 on each dimension. Multiply by the weight. Sum the weighted scores.

### Step 3: Reality-Check the Winner

The spreadsheet will produce a winner. Before you trust it, ask:

- **Does anyone on the team have experience with this broker?** If not, add 3-6 months of learning curve to your project timeline.
- **Is there a managed offering you can start with?** Starting managed and moving self-hosted later is easier than the reverse.
- **Have you talked to someone who runs this in production?** Not a vendor sales engineer. Not a conference speaker. Someone who has been paged at 3 AM because the broker was broken. Find them. Buy them coffee. Ask what they wish they had known before they started.
- **Does the broker's community and governance model give you confidence in its future?** If the broker is maintained by one person or funded by one VC, what happens if that person or that funding disappears?

### Step 4: Run a Proof of Concept

Do not choose a broker based on a spreadsheet. Choose a shortlist based on a spreadsheet, then build a proof of concept with your actual workload, your actual message sizes, your actual consumer patterns, and your actual infrastructure. A two-week POC will teach you more than two months of reading documentation.

---

## Final Advice: The Broker Matters Less Than You Think

After twenty-seven chapters about messaging systems, this may sound like heresy, but it is the most important thing in this book: **the choice of broker matters less than the quality of your architecture and the discipline of your engineering practices.**

A well-designed system with clean event schemas, idempotent consumers, proper error handling, comprehensive observability, and tested failure recovery will work well on *any* competent broker. A poorly designed system with coupled producers, fragile consumers, missing dead-letter handling, and no monitoring will fail on *every* broker, including the most expensive and sophisticated one on the market.

The patterns from Part 1 of this book — schema evolution, delivery guarantees, error handling, observability, testing, anti-patterns — are broker-agnostic. They apply whether you are running Kafka, RabbitMQ, SQS, or carrier pigeons with USB drives taped to their legs (which, for the record, has surprisingly high bandwidth though unacceptable latency). Get those patterns right, and your choice of broker becomes a matter of operational preference rather than architectural survival.

This is not an argument for choosing your broker carelessly. A bad fit will cause real pain — the wrong throughput, the wrong latency, the wrong operational model for your team. But it is an argument against the agonising, months-long broker evaluation process that paralyses some teams. If you have narrowed your shortlist using the frameworks in this chapter, the differences between the finalists are smaller than you think. Pick one. Build on it. Invest your energy in the patterns and practices that matter regardless of which broker is underneath.

The brokers will change. New ones will emerge. Old ones will evolve or fade. Your investment in sound event-driven architecture principles will outlast any specific broker choice. That is the real moat.

---

## A Closing Note: The Future of Event-Driven Architecture

Event-driven architecture is not a trend. It is a structural shift in how we build software, driven by the same forces that drove the shift from monoliths to microservices: the need to build systems that are independently deployable, independently scalable, and resilient to partial failure. Events — facts about things that happened — are the natural interface between autonomous systems. This is not going to un-happen.

What *is* changing, and will continue to change, is how we implement event-driven systems.

**The infrastructure is disappearing into the platform.** Cloud-managed services (Pub/Sub, EventBridge, Event Hubs) are making the broker itself less visible. You publish events and subscribe to them; the cloud provider handles the rest. This trend will accelerate. Within a few years, operating your own message broker will be a deliberate choice made for specific reasons (cost at extreme scale, compliance, latency requirements, multi-cloud), not the default.

**Schemas and contracts are becoming first-class.** The era of publishing arbitrary JSON blobs and hoping consumers can parse them is ending. Schema registries, contract testing, and event catalogs are moving from "nice to have" to "table stakes." AsyncAPI is doing for event-driven interfaces what OpenAPI did for REST APIs. This is unambiguously good.

**Event sourcing and CQRS are moving from niche to mainstream.** The ideas have been around for over a decade, but the tooling is finally catching up. EventStoreDB, Eventuous, Marten, Axon — the ecosystem of purpose-built tools is growing. Event sourcing will not replace CRUD for every use case, but it will become a standard pattern that every senior engineer understands, even if they do not use it daily.

**The broker and the database are converging.** Kafka already functions as a database for some workloads (log-compacted topics as materialised views). EventStoreDB is a database that publishes events. Redis is a cache, a database, and a message broker depending on which features you use. The lines between "where data lives" and "how data moves" are blurring, and the next generation of systems will blur them further.

**Edge and IoT are the next frontier.** As computation moves to the edge — devices, gateways, local servers — event-driven patterns need to work across network boundaries that are unreliable, high-latency, and bandwidth-constrained. Protocols like MQTT and Zenoh, and patterns like event mesh and store-and-forward, will become more important as the edge grows.

The fundamentals, though, remain the fundamentals. Events represent facts. Consumers should be idempotent. Schemas should evolve without breaking consumers. Failure is not an edge case; it is a design input. Observability is not optional. These principles were true when this book was started, they are true now, and they will be true when the specific brokers discussed in these pages have been replaced by whatever comes next.

Build systems that are honest about what happened. The rest is plumbing.
