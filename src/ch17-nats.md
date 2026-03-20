# NATS and JetStream

Most message brokers accumulate complexity the way old houses accumulate extensions. A feature here, a subsystem there, and before long you have something that requires a team of three to operate and a configuration file the length of a novella. NATS took the opposite approach. It started simple, stayed simple, and bet that simplicity itself is the feature that matters most. Fifteen years into the experiment, the bet is paying off.

---

## Overview

NATS was created by Derek Collier, who started the project around 2010 while at Apcera (a company that built a cloud platform, was ahead of its time, and no longer exists — as is the tradition). The original implementation was in Ruby, then rewritten in Go in 2012 for performance. The Go heritage is not incidental — NATS embodies Go's philosophy of doing less, doing it well, and fitting in a single binary.

Collier went on to found Synadia, the company that stewards NATS development. Synadia provides a commercial offering (Synadia Cloud and Synadia Control Plane) but the core NATS server is fully open-source under the Apache 2.0 license. The governance model is straightforward: Synadia employs most of the core maintainers, but the project is genuinely open and the community is active, if smaller than Kafka's or RabbitMQ's.

The NATS story has two chapters. Core NATS (the original) is a fire-and-forget pub/sub system with no persistence, no guaranteed delivery, and extraordinary speed. JetStream (added in NATS Server 2.2, released 2021) adds persistence, exactly-once semantics, and stream processing primitives on top of Core NATS. Understanding both — and the boundary between them — is essential.

---

## Architecture

### Core NATS: The Foundation

Core NATS is a publish-subscribe messaging system built on a single idea: subjects. A subject is a string like `orders.placed` or `payments.us.processed`. Publishers send messages to subjects. Subscribers express interest in subjects. The NATS server routes messages from publishers to subscribers based on subject matching. That is it. There is no queue, no log, no persistence. Messages are delivered to connected subscribers in real time. If no subscriber is connected, the message is gone.

This sounds like a limitation, and it is — but it is a *deliberate* limitation. Core NATS optimises for a specific point in the design space: the lowest possible latency with the simplest possible programming model, for workloads where ephemeral messaging is acceptable. Think of it as UDP for application messaging — fast, simple, no guarantees.

**Publish-subscribe** is the basic pattern. A publisher sends to a subject, and all subscribers on that subject receive the message:

```
Publisher → [orders.placed] → Subscriber A
                            → Subscriber B
                            → Subscriber C
```

**Request-reply** is built into the protocol. A requester publishes a message with a reply subject (an inbox), and a responder subscribes to that subject and sends a response. NATS handles the inbox creation and routing:

```go
// Requester
msg, err := nc.Request("orders.validate", orderData, 2*time.Second)

// Responder
nc.Subscribe("orders.validate", func(msg *nats.Msg) {
    result := validateOrder(msg.Data)
    msg.Respond(result)
})
```

This is a natural fit for microservices that need synchronous-style communication over an asynchronous transport. Multiple responders can subscribe to the same subject, and NATS will route each request to one of them (load-balanced), making it a built-in service mesh primitive.

**Queue groups** provide load balancing across multiple instances of a subscriber. Subscribers that join the same queue group receive messages in a distributed fashion — each message goes to exactly one member of the group:

```go
nc.QueueSubscribe("orders.placed", "order-processors", func(msg *nats.Msg) {
    processOrder(msg.Data)
})
```

Queue groups require no server-side configuration. The subscriber simply declares its group membership at subscription time. This is characteristic of NATS's design philosophy: push complexity to the edges, keep the server simple.

### Subject-Based Routing and Wildcards

NATS subjects use dot-separated tokens. The routing model supports two wildcards:

**`*` (single token wildcard)**: Matches exactly one token in the subject hierarchy.

```
orders.*           → matches orders.placed, orders.cancelled
                   → does NOT match orders.us.placed
```

**`>` (multi-token wildcard)**: Matches one or more tokens at the *end* of a subject.

```
orders.>           → matches orders.placed, orders.us.placed,
                     orders.us.east.placed.confirmed
```

Subject-based routing with wildcards is powerful because it lets you build flexible routing topologies without server-side configuration. Want all events from the US East region? Subscribe to `events.us.east.>`. Want all order events regardless of region? Subscribe to `orders.>`. Want a specific event type across all regions? Subscribe to `*.*.orders.placed`. The subject hierarchy *is* your routing table, and it is defined by convention between publishers and subscribers.

This is a different paradigm from Kafka's topics + partitions, RabbitMQ's exchanges + bindings, or EventBridge's rules + patterns. It is simpler, which means it is faster to set up and easier to understand, but it also means you lack the more sophisticated routing features (content-based routing, header-based filtering, priority queues). Whether this is a limitation depends on whether you need those features.

### JetStream: The Persistence Layer

JetStream was built to answer the question "what if NATS messages needed to survive a server restart?" It adds a persistence layer on top of Core NATS without changing the fundamental architecture.

**Streams** are the persistence primitive. A stream captures messages published to one or more subjects and stores them durably. Streams have configurable retention policies:

- **Limits-based**: Retain up to N messages, N bytes, or N time duration. Oldest messages are discarded when limits are exceeded.
- **Interest-based**: Retain messages only while there are active consumers. Once all consumers have acknowledged a message, it can be removed. This is closer to traditional queue semantics.
- **Work queue**: Each message is consumed by exactly one consumer. Once acknowledged, it is removed. This is a single-consumer queue, not a log.

Streams can be stored in memory or on file (with optional compression). File-based streams survive restarts. Memory-based streams provide higher throughput at the cost of durability — the same trade-off Redis makes, but opt-in rather than the default.

**Consumers** are the subscription mechanism for JetStream. Unlike Core NATS subscriptions (which are ephemeral), JetStream consumers maintain state: their position in the stream, their acknowledgement records, and their delivery tracking.

Consumers can be:
- **Durable**: Server-side state survives consumer disconnection. The consumer resumes from where it left off.
- **Ephemeral**: State is discarded when the consumer disconnects.

Consumers can be:
- **Push-based**: The server delivers messages to the consumer as they become available.
- **Pull-based**: The consumer requests messages when ready. Pull consumers are preferred for most use cases because they provide natural backpressure.

**Exactly-once semantics** in JetStream are achieved through a combination of message deduplication at publish time (using a `Nats-Msg-Id` header and a configurable deduplication window) and double acknowledgement at consume time. The double-ack protocol ensures that neither the server nor the client process a message more than once, at the cost of additional round trips.

**Acknowledgement modes** for consumers:

- `AckExplicit`: Consumer must acknowledge each message individually. The safe default.
- `AckAll`: Acknowledging message N implicitly acknowledges all messages before N. Higher throughput, riskier.
- `AckNone`: No acknowledgement required. Messages are considered delivered when sent. Fire-and-forget.

### Clustering and Super-Clusters

NATS clustering is based on Raft consensus. A cluster of three or more NATS servers provides high availability and fault tolerance. JetStream data is replicated across cluster members (configurable replication factor of 1, 2, or 3).

**Super-clusters** connect multiple NATS clusters across regions or data centres using **gateway connections**. Gateways route messages between clusters transparently — a subscriber in Cluster A receives messages published in Cluster B without any application-level awareness. This creates a global messaging fabric with local-first performance (messages stay local when possible, cross the gateway only when needed).

**Leaf nodes** are NATS servers that connect to a cluster as a client rather than a full member. They extend the messaging fabric to edge locations, remote sites, or isolated environments without the overhead of full cluster membership. A leaf node in a factory floor, a retail store, or an IoT gateway can participate in the global NATS subject space while maintaining local autonomy.

```
                    ┌─────────────────┐
                    │  Super-Cluster   │
                    │                 │
   ┌──────┐        │  ┌──────┐      │        ┌──────┐
   │Cluster│◄──────►│  │Cluster│     │◄──────►│Cluster│
   │  US   │ Gateway│  │  EU  │     │ Gateway│  APAC │
   └──┬───┘        │  └──────┘      │        └──┬───┘
      │             └─────────────────┘           │
      │                                           │
   ┌──┴───┐                                   ┌──┴───┐
   │ Leaf  │                                   │ Leaf  │
   │ Node  │                                   │ Node  │
   │(Edge) │                                   │(Edge) │
   └──────┘                                   └──────┘
```

The leaf node architecture is NATS's genuine competitive advantage for edge computing. Deploying a 20 MB binary to an edge device that automatically connects to the nearest cluster, participates in subject-based routing, and gracefully handles disconnection and reconnection — this is something no other broker does as elegantly.

### NATS KV and NATS Object Store

JetStream's persistence layer enables two additional abstractions:

**NATS KV (Key-Value store)** provides a distributed key-value store built on JetStream streams. It supports put, get, delete, watch (real-time notifications on key changes), and history (retrieve previous values of a key). It is not a replacement for Redis or etcd, but it is useful for configuration distribution, feature flags, and service discovery within the NATS ecosystem. One less dependency.

**NATS Object Store** provides storage for large objects (files, binaries, anything that does not fit in a single NATS message). Objects are chunked and stored in a JetStream stream. This is useful for distributing configuration files, ML models, or other artefacts through the same infrastructure that handles your messaging.

Both features follow NATS's philosophy: if you already have NATS, you should not need a separate system for these common patterns.

---

## Strengths

**Simplicity is the product.** NATS is a single binary (the `nats-server`). The configuration file for a basic cluster fits on a screen. The client libraries are small and well-documented. The concept count is low: subjects, publish, subscribe, request, reply. JetStream adds streams, consumers, and acknowledgements. That is the entire vocabulary. Compare this to Kafka's brokers, ZooKeeper (or KRaft), topics, partitions, consumer groups, offsets, ISR, replication factors, segment files, log compaction, and the extensive configuration surface — NATS is refreshingly minimal.

**Tiny footprint.** The NATS server binary is roughly 20 MB. It starts in milliseconds. It runs on a Raspberry Pi. It runs in a 32 MB container. The memory footprint under moderate load is measured in tens of megabytes. Kafka requires a JVM, gigabytes of heap, and a meaningful amount of disk I/O infrastructure. NATS requires almost nothing. For edge computing, IoT, and resource-constrained environments, this is not a nice-to-have — it is a hard requirement.

**Incredible latency.** Core NATS message delivery is measured in microseconds on a local network. JetStream adds disk I/O and replication overhead, but end-to-end latency is still typically sub-millisecond for persisted messages. NATS consistently benchmarks as one of the fastest messaging systems available.

**Leaf node architecture.** The ability to extend the NATS mesh to edge locations with lightweight leaf nodes is architecturally elegant and practically useful. A leaf node at the edge publishes to local subjects, which are transparently routed to the cloud cluster. The edge node works independently during network partitions and reconnects seamlessly. This is a genuine differentiator.

**Multi-tenancy built in.** NATS accounts provide native multi-tenancy with subject-level isolation, import/export between accounts, and resource limits (connections, data, subscriptions). This is built into the server, not bolted on. For SaaS platforms and shared infrastructure, this reduces the need for separate broker deployments per tenant.

**Single binary deployment.** Download, configure, run. No JVM. No ZooKeeper. No additional dependencies. The upgrade process is "replace the binary and restart." The operational overhead is, in absolute terms, the lowest of any production-grade message broker.

---

## Weaknesses

**Smaller ecosystem.** Kafka has hundreds of connectors, a stream processing framework (Kafka Streams), a SQL interface (ksqlDB), a schema registry, and a massive community producing blog posts, conference talks, and Stack Overflow answers. NATS has good client libraries for many languages and a growing tool ecosystem (the `nats` CLI is excellent), but the breadth of third-party integrations, monitoring tools, and community knowledge is significantly smaller. When you hit a novel problem with Kafka, someone has probably written a blog post about it. With NATS, you may be writing that blog post yourself.

**JetStream maturity.** JetStream was released in 2021. It is stable and production-ready, but it has not had the decade-plus of battle-testing that Kafka's persistence layer has. Edge cases in consumer acknowledgement, cluster failover, and stream recovery are still being discovered and fixed. The pace of improvement is fast, but "fast improvement" implies there were things to improve.

**Community size vs Kafka.** The NATS community is passionate and helpful, but it is orders of magnitude smaller than Kafka's. This affects hiring (fewer engineers know NATS), consulting (fewer firms specialise in it), and tooling (fewer third-party monitoring and management tools).

**No built-in stream processing.** NATS does not have a Kafka Streams equivalent. If you need windowed aggregations, stream joins, or complex event processing, you need an external framework (Flink, Benthos, custom code). JetStream provides the storage and delivery guarantees, but the processing logic is your responsibility.

**No log compaction.** Like Event Hubs and Pub/Sub, NATS JetStream does not support log compaction. The NATS KV store provides latest-value-per-key semantics, but it is not the same as Kafka's compacted topics, which maintain a full changelog of a table.

**Message size limits.** The default maximum message size is 1 MB (configurable up to 64 MB, though large messages are not what NATS is optimised for). For large payloads, you need the Object Store or a claim-check pattern.

**JetStream resource planning.** While Core NATS is effectively "just run it," JetStream requires thinking about storage capacity, replication factors, and retention policies. A JetStream stream with replication factor 3 on file storage consumes three times the disk space. This is expected, but it means JetStream is not quite as "zero planning" as Core NATS.

---

## Operational Reality

This is the section where NATS shines brightest, and it is worth lingering on because operational simplicity is NATS's core value proposition.

**Deployment** is downloading a binary and running it. On Kubernetes, the NATS Helm chart creates a StatefulSet, a headless Service, and a ConfigMap. That is the entire deployment. Compare this to the Kafka Helm chart, which creates ZooKeeper (or KRaft controllers), brokers, PersistentVolumeClaims, ConfigMaps, headless Services, and optionally a schema registry, Connect workers, and a REST proxy.

**Configuration** is a single file. A production NATS cluster configuration is roughly 50 lines. Here is a representative example:

```hcl
# nats-server.conf
listen: 0.0.0.0:4222

jetstream {
  store_dir: /data/jetstream
  max_mem: 1G
  max_file: 100G
}

cluster {
  name: production
  listen: 0.0.0.0:6222
  routes: [
    nats-route://nats-0:6222
    nats-route://nats-1:6222
    nats-route://nats-2:6222
  ]
}

accounts {
  ORDERS {
    jetstream: enabled
    users: [{ user: orders_svc, password: $ORDERS_PASSWORD }]
  }
}
```

**Monitoring** is excellent for a project this size. The NATS server exposes a monitoring endpoint (HTTP on port 8222 by default) with JSON endpoints for connections, routes, gateways, JetStream, and health. The `nats` CLI tool provides real-time dashboards, stream inspection, and consumer management. Prometheus exporters exist and work well. The information you need to assess cluster health is readily available without custom tooling.

**Upgrades** are rolling restarts. Replace the binary, restart each node in sequence. JetStream streams are replicated, so a single-node restart does not cause data unavailability (with replication factor ≥ 2). The NATS team maintains excellent backward compatibility — the protocol has been stable for years.

**Troubleshooting** benefits from the simplicity of the system. When something is wrong, the surface area of possible causes is small. The server logs are clear. The `nats` CLI can inspect subjects, streams, consumers, and cluster state interactively. The mental model fits in your head, which means debugging fits in your head. This is an underappreciated property. The fastest incident resolution comes not from the best tooling, but from the smallest gap between "something is wrong" and "I understand what could be wrong."

**Scaling** follows different patterns for Core NATS and JetStream:
- Core NATS scales by adding servers to the cluster. Message routing is distributed, and the cluster handles millions of messages per second.
- JetStream scales by distributing streams across cluster members. Each stream has a leader that handles writes, so write throughput for a single stream is bounded by a single server's I/O. For higher throughput, partition across multiple streams (application-level sharding).

---

## Ideal Use Cases

**Edge computing and IoT.** The leaf node architecture, tiny footprint, and built-in reconnection handling make NATS the natural choice for edge-to-cloud messaging. A 20 MB binary on an edge gateway that publishes sensor data to `telemetry.factory.line3.temperature` and receives commands on `commands.factory.line3.>` — this is NATS's happy path.

**Kubernetes-native microservices.** NATS runs beautifully in Kubernetes. The StatefulSet deployment is minimal, the resource requirements are low, and the subject-based routing model maps naturally to service communication patterns. For teams that want an in-cluster message bus without the operational weight of Kafka, NATS is the answer.

**Request-reply service communication.** NATS's built-in request-reply pattern provides a lightweight alternative to HTTP-based service-to-service communication. It is faster than REST, simpler than gRPC (no code generation, no proto files), and naturally load-balanced across service instances through queue groups.

**Real-time messaging and signalling.** Chat systems, presence updates, live dashboards, gaming backends — any use case where sub-millisecond message delivery matters and message loss is tolerable (or handled at the application level). Core NATS is purpose-built for this.

**Multi-region and multi-cloud messaging.** The super-cluster and gateway architecture makes NATS one of the simplest options for building a global messaging fabric. Each region runs its own cluster, gateways connect them, and messages route transparently.

**Lightweight event-driven architectures.** For teams that do not need Kafka's throughput or storage capabilities but want more than a REST-based architecture, NATS + JetStream provides event-driven communication with persistence and delivery guarantees at a fraction of the operational cost.

---

## Code Examples

### Go

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"os/signal"
	"time"

	"github.com/nats-io/nats.go"
	"github.com/nats-io/nats.go/jetstream"
)

// --- Core NATS: Pub/Sub and Request-Reply ---

func coreNATSExample() {
	nc, err := nats.Connect(nats.DefaultURL)
	if err != nil {
		log.Fatal(err)
	}
	defer nc.Close()

	// Simple pub/sub
	nc.Subscribe("orders.>", func(msg *nats.Msg) {
		fmt.Printf("Received on %s: %s\n", msg.Subject, string(msg.Data))
	})

	// Queue group subscription (load-balanced across group members)
	nc.QueueSubscribe("orders.placed", "order-processors", func(msg *nats.Msg) {
		fmt.Printf("Processing order: %s\n", string(msg.Data))
	})

	// Request-reply
	nc.Subscribe("orders.validate", func(msg *nats.Msg) {
		// Responder: validate the order and reply
		msg.Respond([]byte(`{"valid": true}`))
	})

	// Requester: send a request and wait for a reply
	resp, err := nc.Request("orders.validate",
		[]byte(`{"orderId": "ord-7829"}`),
		2*time.Second,
	)
	if err != nil {
		log.Printf("Request failed: %v", err)
	} else {
		fmt.Printf("Validation response: %s\n", string(resp.Data))
	}

	// Publish (fire-and-forget)
	order := map[string]interface{}{
		"orderId":     "ord-7829",
		"totalAmount": 149.99,
		"region":      "us-east-1",
	}
	data, _ := json.Marshal(order)
	nc.Publish("orders.placed", data)
}

// --- JetStream: Persistent Streams and Consumers ---

func jetStreamExample() {
	nc, err := nats.Connect(nats.DefaultURL)
	if err != nil {
		log.Fatal(err)
	}
	defer nc.Close()

	js, err := jetstream.New(nc)
	if err != nil {
		log.Fatal(err)
	}
	ctx := context.Background()

	// Create or update a stream
	stream, err := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
		Name:      "ORDERS",
		Subjects:  []string{"orders.>"},
		Storage:   jetstream.FileStorage,
		Replicas:  3,
		Retention: jetstream.LimitsPolicy,
		MaxAge:    7 * 24 * time.Hour, // 7 days retention
		MaxBytes:  10 * 1024 * 1024 * 1024, // 10 GB
		Discard:   jetstream.DiscardOld,
	})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Stream %s: %d messages\n", stream.CachedInfo().Config.Name,
		stream.CachedInfo().State.Msgs)

	// Publish with acknowledgement
	ack, err := js.Publish(ctx, "orders.placed", data,
		jetstream.WithMsgID("ord-7829-placed"), // For deduplication
	)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Published to stream %s, seq %d\n", ack.Stream, ack.Sequence)

	// Create a durable pull consumer
	consumer, err := js.CreateOrUpdateConsumer(ctx, "ORDERS", jetstream.ConsumerConfig{
		Durable:       "order-processor",
		AckPolicy:     jetstream.AckExplicitPolicy,
		FilterSubject: "orders.placed",
		MaxDeliver:    5,           // Max redelivery attempts
		AckWait:       30 * time.Second,
		DeliverPolicy: jetstream.DeliverAllPolicy,
	})
	if err != nil {
		log.Fatal(err)
	}

	// Consume messages
	iter, err := consumer.Messages()
	if err != nil {
		log.Fatal(err)
	}

	// Graceful shutdown
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, os.Interrupt)

	go func() {
		for {
			msg, err := iter.Next()
			if err != nil {
				log.Printf("Consumer error: %v", err)
				return
			}

			var order map[string]interface{}
			if err := json.Unmarshal(msg.Data(), &order); err != nil {
				log.Printf("Invalid message: %v", err)
				msg.Term() // Terminate — do not redeliver
				continue
			}

			fmt.Printf("Processing order %s (attempt %d)\n",
				order["orderId"],
				msg.Headers().Get("Nats-Num-Delivered"))

			if err := processOrder(order); err != nil {
				msg.Nak() // Negative ack — redeliver
			} else {
				msg.Ack() // Success
			}
		}
	}()

	<-sigCh
	iter.Stop()
	fmt.Println("Shut down gracefully")
}

func processOrder(order map[string]interface{}) error {
	fmt.Printf("Order %s: $%.2f\n", order["orderId"], order["totalAmount"])
	return nil
}
```

### Python (nats-py)

```python
import asyncio
import json
import signal
import nats
from nats.js.api import StreamConfig, ConsumerConfig, RetentionPolicy, AckPolicy

async def main():
    nc = await nats.connect("nats://localhost:4222")
    js = nc.jetstream()

    # --- Create a stream ---
    await js.add_stream(
        StreamConfig(
            name="ORDERS",
            subjects=["orders.>"],
            retention=RetentionPolicy.LIMITS,
            max_age=7 * 24 * 60 * 60 * 1_000_000_000,  # 7 days in nanoseconds
            max_bytes=10 * 1024 * 1024 * 1024,           # 10 GB
            num_replicas=3,
        )
    )

    # --- Publish ---
    order = {
        "orderId": "ord-7829",
        "totalAmount": 149.99,
        "region": "us-east-1",
    }
    ack = await js.publish(
        "orders.placed",
        json.dumps(order).encode(),
        headers={"Nats-Msg-Id": "ord-7829-placed"},  # Deduplication
    )
    print(f"Published to {ack.stream}, seq {ack.seq}")

    # --- Subscribe with durable consumer ---
    sub = await js.pull_subscribe(
        "orders.placed",
        durable="order-processor",
        config=ConsumerConfig(
            ack_policy=AckPolicy.EXPLICIT,
            max_deliver=5,
            ack_wait=30,
        ),
    )

    running = True

    def shutdown():
        nonlocal running
        running = False

    loop = asyncio.get_event_loop()
    loop.add_signal_handler(signal.SIGINT, shutdown)

    while running:
        try:
            messages = await sub.fetch(batch=10, timeout=5)
            for msg in messages:
                try:
                    event = json.loads(msg.data.decode())
                    print(f"Processing order {event['orderId']}: "
                          f"${event['totalAmount']}")

                    await process_order(event)
                    await msg.ack()

                except Exception as e:
                    print(f"Processing failed: {e}")
                    await msg.nak()

        except nats.errors.TimeoutError:
            continue  # No messages available, loop back

    await nc.close()
    print("Consumer shut down")


async def process_order(event: dict):
    pass


# --- Core NATS: Simple pub/sub and request-reply ---
async def core_nats_example():
    nc = await nats.connect("nats://localhost:4222")

    # Subscribe with queue group
    async def handler(msg):
        data = json.loads(msg.data.decode())
        print(f"Received: {data}")

    await nc.subscribe("orders.placed", queue="order-processors", cb=handler)

    # Request-reply
    async def validator(msg):
        order = json.loads(msg.data.decode())
        response = {"valid": True, "orderId": order["orderId"]}
        await msg.respond(json.dumps(response).encode())

    await nc.subscribe("orders.validate", cb=validator)

    response = await nc.request(
        "orders.validate",
        json.dumps({"orderId": "ord-7829"}).encode(),
        timeout=2.0,
    )
    print(f"Validation: {response.data.decode()}")

    # Publish
    await nc.publish(
        "orders.placed",
        json.dumps({"orderId": "ord-7829", "total": 149.99}).encode(),
    )
    await asyncio.sleep(1)  # Let the subscriber process
    await nc.close()


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Verdict

NATS is the message broker for people who have operated other message brokers and are tired. Tired of JVM tuning. Tired of ZooKeeper. Tired of configuration files that require a manual. Tired of "simple" deployments that involve twelve Helm charts and a prayer.

That sounds like damning-with-faint-praise, but it is the highest compliment you can pay an infrastructure component. The best infrastructure is the infrastructure you do not think about, and NATS comes closer to that ideal than any other message broker in this book. It deploys in minutes, runs on resources that Kafka would consider a rounding error, and provides a programming model that a junior engineer can learn in an afternoon.

JetStream elevates NATS from "interesting but limited" to "serious contender." The addition of persistence, exactly-once semantics, and stream processing primitives means NATS can serve as the primary messaging layer for many workloads that previously required Kafka. It is not a Kafka replacement for all use cases — Kafka's throughput at scale, ecosystem breadth, and log compaction are not matched — but for the majority of teams whose event throughput is measured in thousands or tens of thousands per second rather than millions, NATS provides the same guarantees with dramatically less operational overhead.

The leaf node architecture is a genuine innovation. For organisations with edge computing requirements — IoT, retail, manufacturing, distributed offices — NATS provides a unified messaging fabric from edge to cloud that no other broker matches in simplicity or footprint.

The honest risk assessment: NATS is a smaller project with a smaller community. If you need a connector for every database, a managed offering on every cloud, and the confidence that comes from hundreds of conference talks about production deployments, Kafka is the safer bet. If you need a message broker that you can deploy, understand, operate, and debug with a small team, NATS is the bet worth making.

The best technology choice is the one that fits your team, your workload, and your tolerance for operational complexity. For a growing number of teams, that choice is NATS — not because it does the most, but because it demands the least while delivering what matters.
