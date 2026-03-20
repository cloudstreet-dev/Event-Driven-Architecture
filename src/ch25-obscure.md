# The Obscure and the Curious

The previous chapters covered the brokers that dominate the conversation — the ones that show up on every "Top 5 Message Brokers" listicle, the ones with conference talks and certification programmes and vendor booths the size of studio apartments. But the messaging landscape is considerably wider than the conference circuit suggests. There are tools that solve specific problems brilliantly, tools that approach the problem from an entirely different angle, and tools that most engineers have never heard of despite being quietly excellent.

This chapter is for them. Some are brokers. Some are libraries. Some are somewhere in between. All of them deserve a look, even if most will never be the centrepiece of your architecture. The messaging world has a long tail, and the long tail is where interesting things happen.

---

## QStash (Upstash)

### Serverless Messaging for People Who Do Not Want to Run Anything

QStash is what happens when you take the concept of a message queue and strip it down to an HTTP API with a credit card attached. Built by Upstash, the company that also offers serverless Redis and Kafka, QStash is an HTTP-based message queue designed specifically for serverless and edge function environments. You POST a message to QStash with a destination URL. QStash delivers it. If the destination fails, QStash retries. That is, conceptually, the entire product.

The insight behind QStash is that serverless functions (AWS Lambda, Cloudflare Workers, Vercel Edge Functions) have an awkward relationship with traditional message brokers. A Lambda function cannot maintain a persistent TCP connection to Kafka. It spins up, does its work, and dies. Traditional consumer patterns — long-polling, persistent connections, consumer group coordination — are fundamentally at odds with the serverless execution model. QStash solves this by inverting the flow: instead of the consumer pulling messages, QStash pushes messages to the consumer via HTTP. Your serverless function is just an HTTP endpoint that receives POST requests.

QStash includes built-in delay scheduling (deliver this message in 30 minutes), automatic retries with configurable backoff, deduplication, and basic dead-letter handling. It also supports cron-like scheduling, making it a lightweight alternative to EventBridge Scheduler or Cloud Scheduler for simple periodic tasks. The pricing is per-message, which aligns naturally with serverless cost models — you pay when work happens, not when infrastructure idles.

The limitations are exactly what you would expect. Throughput is modest — this is not a tool for streaming a million events per second. There are no consumer groups, no partitioning, no ordering guarantees beyond single-message delivery. The delivery model is push-only, so your consumer must be an HTTP endpoint, which means you need something publicly addressable or tunnelled. And the entire thing is a managed service with no self-hosted option — you are trusting Upstash with your message delivery and accepting the vendor dependency. For the use cases it targets — background jobs, webhooks, scheduled tasks, inter-service communication in serverless architectures — these limitations are perfectly acceptable. For anything else, you probably want a real broker.

```bash
# Publish a message to QStash — deliver to your endpoint with retry
curl -X POST "https://qstash.upstash.io/v2/publish/https://my-api.example.com/webhook" \
  -H "Authorization: Bearer <QSTASH_TOKEN>" \
  -H "Content-Type: application/json" \
  -H "Upstash-Delay: 60s" \
  -H "Upstash-Retries: 3" \
  -d '{"orderId": "12345", "action": "process_payment"}'

# Schedule a recurring message (cron)
curl -X POST "https://qstash.upstash.io/v2/schedules" \
  -H "Authorization: Bearer <QSTASH_TOKEN>" \
  -H "Content-Type: application/json" \
  -H "Upstash-Cron: */5 * * * *" \
  -d '{"destination": "https://my-api.example.com/cleanup", "body": "{}"}'
```

---

## Watermill (Go)

### Not a Broker — a Way of Thinking About Messages

Watermill is an event-driven library for Go, and the most important thing to understand is what it is *not*: it is not a message broker. It does not store messages. It does not manage subscriptions. It does not replicate data. It is a library that provides a clean, consistent abstraction over *other* systems that do those things. You plug in Kafka, RabbitMQ, Google Pub/Sub, NATS, Amazon SQS, or even an in-memory channel as the backend, and Watermill gives you a uniform API for publishing, subscribing, and routing messages.

The core value proposition is middleware. Watermill borrows the middleware pattern from HTTP frameworks (think Go's `net/http` middleware or Express.js middleware) and applies it to message processing. You can chain middleware functions that handle retries, deduplication, logging, metrics, tracing, poison message detection, and throttling — all independent of the underlying broker. This is genuinely useful. If you have ever written the same "retry with exponential backoff and dead-letter on exhaustion" logic for the fourth time across three different broker integrations, Watermill's middleware chains will feel like relief.

The router is the other key concept. Instead of writing bare consumer loops, you define routes that bind a subscriber topic, a handler function, and an optional publisher topic. The router manages the lifecycle — starting subscribers, passing messages through middleware, calling your handler, and optionally publishing the result. It handles graceful shutdown, which is the kind of thing that sounds trivial until you have debugged a production system that loses messages because `os.Exit` was called while a handler was mid-transaction.

Watermill is opinionated about structure but agnostic about infrastructure, which is an uncommon and valuable position. The main risk is the usual risk of abstraction layers: you lose access to broker-specific features. If you need Kafka's exactly-once transactions, or RabbitMQ's exchange topologies, or NATS's subject-based addressing with wildcards, the Watermill abstraction may not expose them. For many applications, the features Watermill *does* expose are sufficient. For others, the abstraction leaks at exactly the wrong moment. Know your use case before committing.

```go
package main

import (
    "context"
    "log"

    "github.com/ThreeDotsLabs/watermill"
    "github.com/ThreeDotsLabs/watermill-kafka/v3/pkg/kafka"
    "github.com/ThreeDotsLabs/watermill/message"
    "github.com/ThreeDotsLabs/watermill/message/router/middleware"
)

func main() {
    logger := watermill.NewStdLogger(false, false)

    subscriber, _ := kafka.NewSubscriber(
        kafka.SubscriberConfig{
            Brokers:       []string{"localhost:9092"},
            ConsumerGroup: "order-processor",
        },
        logger,
    )

    publisher, _ := kafka.NewPublisher(
        kafka.PublisherConfig{Brokers: []string{"localhost:9092"}},
        logger,
    )

    router, _ := message.NewRouter(message.RouterConfig{}, logger)

    // Middleware chains — the real value of Watermill
    router.AddMiddleware(
        middleware.Retry{MaxRetries: 3}.Middleware,
        middleware.Recoverer,
        middleware.CorrelationID,
    )

    router.AddHandler(
        "order_to_invoice",       // handler name
        "orders",                 // subscribe topic
        subscriber,
        "invoices",               // publish topic
        publisher,
        func(msg *message.Message) ([]*message.Message, error) {
            log.Printf("Processing order: %s", string(msg.Payload))
            invoice := message.NewMessage(watermill.NewUUID(), msg.Payload)
            return []*message.Message{invoice}, nil
        },
    )

    router.Run(context.Background())
}
```

---

## Eventuous

### Event Sourcing for .NET, Without the Archaeology

Eventuous is an opinionated event sourcing library for .NET. If you are building event-sourced systems on the .NET platform and you have spent time evaluating Marten, Axon (via the Java interop pain), or rolling your own aggregate/event/projection infrastructure for the third time, Eventuous deserves your attention.

The library is designed to work with EventStoreDB (covered separately below) as its primary event store, though it supports other backends. Eventuous provides the wiring that sits between your domain model and the event store: aggregate base classes, command handling, event serialisation, subscriptions, projections (read model updates), and gateway patterns for integrating with external systems. It is opinionated in the sense that it steers you toward specific patterns — aggregates that emit events, command handlers that load and save aggregates, subscriptions that project events into read models — rather than giving you a toolkit of primitives and wishing you luck.

The opinionation is a feature, not a bug, for teams that have decided they are doing event sourcing and want to get to productive code quickly rather than spending their first sprint debating whether aggregates should be classes or records, whether events should be interfaces or sealed hierarchies, and whether the command handler should be a method on the aggregate or a separate service. Eventuous makes these decisions for you. If you agree with the decisions, you move fast. If you disagree, you will fight the framework, and fighting frameworks is a losing proposition.

The integration with EventStoreDB is where Eventuous is strongest. Subscriptions — both catch-up subscriptions (replaying the event stream from a position) and persistent subscriptions (server-managed consumer positions) — are first-class concepts. Projections can be built using either EventStoreDB's built-in projection engine or Eventuous's own subscription-based projection infrastructure, which projects events into MongoDB, Elasticsearch, PostgreSQL, or other read stores. For teams building CQRS/ES systems on .NET with EventStoreDB, Eventuous is likely the fastest path from "we have decided to do event sourcing" to "we are shipping features."

```csharp
// Define domain events
public record RoomBooked(string RoomId, string GuestName, DateTime CheckIn, DateTime CheckOut);
public record BookingCancelled(string Reason);

// Aggregate with event sourcing
public class Booking : Aggregate<BookingState> {
    public void BookRoom(string roomId, string guest, DateTime checkIn, DateTime checkOut) {
        EnsureDoesntExist();
        Apply(new RoomBooked(roomId, guest, checkIn, checkOut));
    }

    public void Cancel(string reason) {
        EnsureExists();
        Apply(new BookingCancelled(reason));
    }
}

public record BookingState : State<BookingState> {
    public string RoomId { get; init; }
    public bool IsCancelled { get; init; }

    public BookingState() {
        On<RoomBooked>((state, evt) => state with { RoomId = evt.RoomId });
        On<BookingCancelled>((state, _) => state with { IsCancelled = true });
    }
}

// Command handler — Eventuous wires this up
public class BookingCommandService : CommandService<Booking, BookingState, BookingId> {
    public BookingCommandService(IAggregateStore store) : base(store) {
        OnNew<BookRoom>(cmd => new BookingId(cmd.BookingId),
            (booking, cmd) => booking.BookRoom(cmd.RoomId, cmd.Guest, cmd.CheckIn, cmd.CheckOut));

        OnExisting<CancelBooking>(cmd => new BookingId(cmd.BookingId),
            (booking, cmd) => booking.Cancel(cmd.Reason));
    }
}
```

---

## Mochi MQTT

### When You Need MQTT but Mosquitto Feels Like Overkill

MQTT is the lingua franca of IoT messaging — lightweight, low-bandwidth, designed for devices that may have the processing power of a potato. The dominant open-source MQTT broker is Eclipse Mosquitto, which is excellent and battle-tested but is also a standalone daemon written in C that you deploy as infrastructure. Mochi MQTT takes a different approach: it is an embeddable MQTT broker written in Go that you can import as a library and run inside your own application.

The use case is specific but not rare. You are building a Go application — perhaps an IoT gateway, an edge computing service, or a testing harness — and you need MQTT broker functionality without deploying a separate process. Maybe you want to embed MQTT message handling directly in your application server. Maybe you are building an appliance or a self-contained system where minimising process count matters. Maybe you just want an MQTT broker you can spin up in a test with `go test` and tear down when the test finishes, without Docker or process management.

Mochi MQTT implements MQTT v5.0 (and v3.1.1) with support for QoS 0, 1, and 2, retained messages, will messages, topic filters with wildcards, shared subscriptions, and the other features you expect from an MQTT broker. It supports pluggable persistence backends — in-memory for testing, Bolt or Badger for embedded persistence, or you can write your own. It also supports pluggable authentication via hooks, so you can integrate it with your application's existing auth system rather than managing a separate credential store.

The trade-off is clear: Mochi MQTT is not Mosquitto. It does not have Mosquitto's years of production hardening, its bridging capabilities, or its ecosystem of plugins. For a fleet of ten thousand devices in production, you probably want Mosquitto (or EMQX, or HiveMQ, or a managed MQTT service). For embedding broker functionality in a Go application, for testing, for edge deployments, or for situations where "deploy another daemon" is not an option, Mochi is a clean and well-designed solution.

```go
package main

import (
    "log"
    "os"
    "os/signal"
    "syscall"

    mqtt "github.com/mochi-mqtt/server/v2"
    "github.com/mochi-mqtt/server/v2/hooks/auth"
    "github.com/mochi-mqtt/server/v2/listeners"
)

func main() {
    // Create the broker — it is just a Go struct
    server := mqtt.New(&mqtt.Options{
        InlineClient: true, // Allow the embedding app to subscribe/publish
    })

    // Allow all connections (use a real auth hook in production)
    _ = server.AddHook(new(auth.AllowHook), nil)

    // Add a TCP listener
    tcp := listeners.NewTCP(listeners.Config{
        ID:      "tcp1",
        Address: ":1883",
    })
    _ = server.AddListener(tcp)

    // Subscribe from within the embedding application
    callbackFn := func(cl *mqtt.Client, sub mqtt.Subscription, pk mqtt.Packet) {
        log.Printf("Embedded subscriber received on %s: %s",
            pk.TopicName, string(pk.Payload))
    }
    _ = server.Subscribe("sensors/+/temperature", 1, callbackFn)

    go func() { _ = server.Serve() }()
    log.Println("MQTT broker running on :1883")

    // Graceful shutdown
    sig := make(chan os.Signal, 1)
    signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
    <-sig
    _ = server.Close()
}
```

---

## LavinMQ

### RabbitMQ's Diet Cousin, Written in Crystal

LavinMQ is an AMQP 0.9.1 compatible message broker written in Crystal — yes, Crystal, the language that looks like Ruby but compiles to native code. It is wire-compatible with RabbitMQ, meaning your existing RabbitMQ client libraries (for any language) work with LavinMQ without modification. The pitch is simple: all the protocol compatibility of RabbitMQ with a dramatically smaller resource footprint.

And "dramatically smaller" is not marketing hyperbole. LavinMQ consistently runs at a fraction of RabbitMQ's memory usage. Where a RabbitMQ node might consume several gigabytes of RAM under moderate load, LavinMQ can handle comparable workloads in hundreds of megabytes. The disk I/O profile is also leaner — LavinMQ uses memory-mapped files and an append-only segment-based storage engine that avoids the complex Erlang queue mirroring machinery. For environments where resources are constrained — edge deployments, small VPS instances, development machines, or situations where you genuinely do not need the full weight of RabbitMQ's feature set — the resource savings are meaningful.

The trade-offs are significant and worth understanding. LavinMQ does not implement RabbitMQ's quorum queues, shovel plugin, federation, or the more advanced clustering features. It does not have RabbitMQ's plugin ecosystem. The community is small. The Crystal language ecosystem, while growing, is nowhere near the size of Erlang/OTP's, which means fewer contributors and a smaller pool of people who can debug the internals. If you need RabbitMQ's full feature set, use RabbitMQ. LavinMQ is for situations where you need the AMQP protocol with minimal overhead, and you can live without the features you are giving up.

LavinMQ is developed by 84codes, the company behind CloudAMQP (a major managed RabbitMQ provider), which means the team building it understands AMQP in production at scale. This is reassuring — they are not building a toy, they are building a tool they understand the need for from years of operating RabbitMQ for thousands of customers.

---

## Apache RocketMQ

### The Messaging Giant You Have Probably Never Used

Apache RocketMQ is a distributed messaging system originally developed at Alibaba, donated to the Apache Software Foundation, and widely deployed across Alibaba's infrastructure where it handles trillions of messages. It is one of the most battle-tested messaging systems in existence, but unless you work with Chinese technology companies or read Chinese-language technical blogs, you may have never encountered it.

RocketMQ occupies a similar space to Kafka — distributed, partitioned, high-throughput, log-based — but with a different set of design decisions. The most distinctive feature is transaction messages: RocketMQ has first-class support for a two-phase commit protocol that coordinates message publishing with local database transactions. You begin a "half message" (invisible to consumers), execute your local transaction, and then commit or roll back the half message based on whether the local transaction succeeded. If the commit/rollback is lost (process crash, network failure), RocketMQ will call back to your application to check the transaction status. This is a feature that Kafka users typically implement at the application level with the Outbox pattern; RocketMQ builds it into the broker.

Other notable features include scheduled messages with arbitrary delay (not just fixed delay levels, though the implementation details have evolved across versions), message filtering on the broker side using SQL92-like expressions or tag-based filtering, and a built-in tracing and metrics system. The operational model uses a "NameServer" for service discovery (simpler than ZooKeeper but less feature-rich) and supports both master-slave and Raft-based replication in newer versions.

The adoption barrier outside China is real and worth acknowledging honestly. Documentation quality in English has historically been uneven. The community discussion happens substantially in Chinese. Client library quality varies by language — the Java client is excellent (it is what Alibaba uses), while clients for other languages range from adequate to experimental. If you are a Java shop comfortable reading some Chinese-language resources and you need transaction message support without implementing it yourself, RocketMQ is a serious option. If you are a polyglot team that relies on English-language documentation and Stack Overflow, the friction will be higher than with Kafka or RabbitMQ.

```java
// RocketMQ transaction message — the killer feature
TransactionMQProducer producer = new TransactionMQProducer("tx-producer-group");
producer.setNamesrvAddr("localhost:9876");
producer.setTransactionListener(new TransactionListener() {

    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        // This runs after the half message is sent but before it's visible
        try {
            orderRepository.save(new Order(msg.getKeys(), msg.getBody()));
            return LocalTransactionState.COMMIT_MESSAGE;  // Make message visible
        } catch (Exception e) {
            return LocalTransactionState.ROLLBACK_MESSAGE; // Discard the message
        }
    }

    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        // RocketMQ calls this if commit/rollback was lost
        // Check your database: did the order actually save?
        boolean exists = orderRepository.existsByOrderId(msg.getKeys());
        return exists
            ? LocalTransactionState.COMMIT_MESSAGE
            : LocalTransactionState.ROLLBACK_MESSAGE;
    }
});
producer.start();

Message msg = new Message("orders", "OrderCreated", orderId,
    orderJson.getBytes(StandardCharsets.UTF_8));
producer.sendMessageInTransaction(msg, null);
```

---

## EventStoreDB

### The Event Store That Started the Conversation

EventStoreDB is the database that Greg Young built to prove that event sourcing was not just an academic exercise but a practical architecture. If event sourcing has a spiritual home, EventStoreDB is it. While you *can* do event sourcing on top of PostgreSQL, Kafka, or DynamoDB (and many people do), EventStoreDB was purpose-built for the pattern, and that purpose-built nature shows in everything from its data model to its query capabilities.

The core abstraction is the stream — an ordered, append-only sequence of events identified by a stream name. You write events to streams (typically one stream per aggregate: `order-12345`, `customer-67890`). You read events from streams, either forward from a position or backward from the end. This maps directly to the event sourcing pattern: to reconstitute an aggregate, read its stream from the beginning and replay the events. To see what happened to a specific entity, read its stream. Simple.

Where EventStoreDB gets interesting is projections and subscriptions. Projections are server-side JavaScript functions that run over event streams and produce new streams, state, or views. You can create a projection that reads from all `order-*` streams and produces a `high-value-orders` stream containing only orders above a certain amount. Or a projection that maintains a running count of events by type. Projections run continuously as new events are written, making them a form of real-time stream processing built into the database. Subscriptions allow clients to follow streams in real time — your read model updater subscribes to a category of streams and updates a SQL database as new events arrive. Persistent subscriptions add consumer-group-like semantics with server-managed checkpoints.

The operational story has improved significantly over the years. EventStoreDB 20+ moved from the Mono runtime to .NET and introduced a gRPC-based client protocol, which broadened client library support beyond the .NET ecosystem. Clustering uses a gossip-based protocol for leader election and supports both synchronous and asynchronous replication. There is a commercial cloud offering (Event Store Cloud) for teams that prefer managed infrastructure. The community is passionate, knowledgeable, and occasionally evangelical in a way that only people who have had a genuine architectural revelation can be.

The honest assessment: EventStoreDB is exceptional at what it was designed for — event sourcing and CQRS. If your architecture is built around event-sourced aggregates, projections, and read models, EventStoreDB is the most natural fit. If you are doing general-purpose pub/sub, event streaming, or task queuing, you are better served by tools designed for those patterns. EventStoreDB is a specialist, not a generalist, and that is exactly what makes it valuable.

```csharp
// Writing events to EventStoreDB
var client = new EventStoreClient(EventStoreClientSettings.Create("esdb://localhost:2113?tls=false"));

var events = new[] {
    new EventData(
        Uuid.NewUuid(),
        "OrderPlaced",
        JsonSerializer.SerializeToUtf8Bytes(new {
            OrderId = "order-42",
            CustomerId = "cust-7",
            Total = 159.99m,
            PlacedAt = DateTime.UtcNow
        })
    ),
    new EventData(
        Uuid.NewUuid(),
        "OrderConfirmed",
        JsonSerializer.SerializeToUtf8Bytes(new {
            OrderId = "order-42",
            ConfirmedAt = DateTime.UtcNow
        })
    )
};

// Append to a stream — optimistic concurrency via expected revision
await client.AppendToStreamAsync(
    "order-42",
    StreamState.NoStream,  // Expect the stream doesn't exist yet
    events
);

// Read the stream back
var result = client.ReadStreamAsync(Direction.Forwards, "order-42", StreamPosition.Start);
await foreach (var evt in result) {
    Console.WriteLine($"{evt.Event.EventType}: {Encoding.UTF8.GetString(evt.Event.Data.Span)}");
}

// Subscribe to all events in the "order" category
await client.SubscribeToStreamAsync(
    "$ce-order",  // Category projection — all streams starting with "order-"
    FromStream.Start,
    (sub, evt, ct) => {
        Console.WriteLine($"Read model update: {evt.Event.EventType}");
        return Task.CompletedTask;
    }
);
```

---

## Liftbridge

### NATS Plus Persistence, Before JetStream Made It Redundant

Liftbridge is a cautionary tale about timing in open source. It was created to solve a real problem: NATS Core was excellent for ephemeral pub/sub messaging but had no persistence. Messages were fire-and-forget — if no subscriber was listening when a message was published, it was gone. Liftbridge added a persistence layer on top of NATS by implementing a Kafka-like log abstraction — streams with offsets, consumer positions, and durable storage — while using NATS as the transport layer.

The architecture was clever. Liftbridge servers formed a cluster alongside NATS servers, intercepting messages on designated subjects and writing them to persistent logs. Consumers could then read from these logs using offsets, just like Kafka. You got NATS's simplicity and performance for ephemeral messaging, plus Kafka-like durability and replay for the subjects that needed it. It was the best of both worlds — or at least, that was the pitch.

Then NATS JetStream arrived. JetStream, built directly into the NATS server by the core NATS team at Synadia, provided persistence, stream processing, key-value storage, and object storage as a first-party feature. It solved the same fundamental problem as Liftbridge but with deeper integration, official support, and the full weight of the NATS community behind it. Liftbridge, as a third-party add-on solving a problem that the first party had now officially solved, found its reason for existing substantially eroded.

Liftbridge still works. The concepts are sound. If you encounter it in an existing system, there is no urgent reason to rip it out. But for new projects, JetStream is the clear choice for adding persistence to NATS. Liftbridge's story is a useful reminder that building on top of someone else's platform carries inherent risk: the platform may eventually absorb your value proposition. It happens in messaging. It happens in the cloud. It happens everywhere in software. The question is not *whether* it will happen but whether you have shipped enough value before it does.

---

## Notable Mentions

The long tail of messaging systems extends well beyond what any single chapter can cover. Here are a few more worth knowing about, even if they do not warrant a full profile.

**NSQ** — Originally built at Bitly, NSQ is a real-time distributed messaging platform written in Go. It emphasises operational simplicity: no single point of failure, no complex configuration, minimal dependencies. Messages are pushed to consumers, and there are no consumer groups or complex routing — just topics and channels. NSQ was ahead of its time in prioritising developer experience and operational simplicity, and it influenced the design of several later systems. It still works well for straightforward pub/sub workloads where you want something simpler than Kafka but more durable than Redis Pub/Sub. Development has slowed, but the codebase is stable and the design is sound.

**Zenoh** — An interesting protocol and implementation emerging from the Eclipse Foundation (originally from ADLINK Technology, now ZettaScale). Zenoh is a pub/sub/query protocol designed for robotics, IoT, and edge-to-cloud communication. It unifies data in motion (pub/sub), data at rest (storage), and data in computation (queries) under a single protocol. The most intriguing aspect is its ability to bridge different network topologies — it can work peer-to-peer, through routers, or via brokerless gossip. If you are building systems that span edge devices, fog nodes, and cloud services, Zenoh's unified model is worth investigating. The community is small but growing, and the protocol design is genuinely novel rather than "Kafka but different."

**Tributary** — A Python library for building streaming reactive pipelines. It is less a messaging system and more a dataflow framework, using Python's async capabilities to build directed acyclic graphs of computations. Useful for data science workflows that need reactive processing without the overhead of deploying a full streaming platform.

**Memphis.dev's legacy** — Covered in detail in Chapter 21, but worth mentioning here as a member of the "developer experience layer over NATS JetStream" category, which turned out to be a difficult space to sustain a business in.

**KubeMQ** — A Kubernetes-native message broker that runs as a single container and supports multiple messaging patterns (queues, pub/sub, RPC). The pitch is simplicity for teams that want messaging inside Kubernetes without the operational weight of Kafka or RabbitMQ. The community is small, and the long-term viability question applies.

**VerneMQ and NanoMQ** — Alternative MQTT brokers. VerneMQ is Erlang-based (like RabbitMQ) and designed for large-scale IoT deployments. NanoMQ is a lightweight C-based broker from the same organisation behind NNG (nanomsg next generation). Both are worth evaluating if EMQX or Mosquitto do not fit your specific constraints.

**Beanstalkd** — A simple, fast work queue. It does one thing — job queuing with priorities, delays, and time-to-run — and does it well. No pub/sub, no streaming, no log compaction. If all you need is a work queue and you find even Redis overly complex for the task, Beanstalkd's simplicity is appealing. It has been around since 2007 and remains quietly useful.

---

## The Long Tail of Messaging

If this chapter has demonstrated anything, it is that the messaging landscape is far more diverse than the Kafka-versus-RabbitMQ debate suggests. There are hundreds of messaging systems in existence, ranging from battle-tested infrastructure running at Alibaba scale to a single developer's weekend project on GitHub with eleven stars and a README that says "TODO: add documentation."

This diversity is not a problem. It is a sign of a healthy engineering ecosystem. Different problems genuinely require different solutions. An IoT gateway collecting sensor data from ten thousand devices has different needs than a financial trading system processing market data with microsecond latency, and both have different needs than a startup's background job queue running on a single $20/month VPS. A tool that is perfect for one of these is likely terrible for the others.

The risk is not in the diversity itself but in the temptation to chase novelty. Every new messaging system promises to fix the problems of the ones that came before. Sometimes it does. Sometimes the "problems" it fixes are actually trade-offs that exist for good reasons, and the new system has simply chosen different trade-offs that it has not yet had enough production miles to discover. The graveyard of messaging systems that were going to replace Kafka is well-populated and continues to accept new residents.

The practical advice is straightforward. For your core messaging infrastructure, choose something battle-tested with a large community, active maintenance, and a track record measured in years, not months. For specialised use cases — embedded MQTT, event sourcing, serverless job queues, edge computing — the niche tools in this chapter may be exactly what you need, and they may save you from contorting a general-purpose broker into a shape it was never designed for. Know the difference between your core infrastructure choices and your specialised tooling choices, and apply different risk tolerances to each.

The messaging world will continue to diversify. New protocols will emerge. New brokers will be announced. New "Kafka killers" will appear on Hacker News, generate excitement, and either prove their worth or quietly fade. The evaluation framework from Chapter 10 applies to all of them. The fundamentals — durability, ordering, delivery semantics, operational complexity — do not change just because the implementation language is novel or the website is well-designed. Judge tools by what they do under load, not by what they promise in blog posts.
