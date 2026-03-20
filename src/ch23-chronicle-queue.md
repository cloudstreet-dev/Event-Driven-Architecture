# Chronicle Queue

We have now left the territory of general-purpose message brokers and entered the domain of people who measure latency in microseconds and consider garbage collection a personal affront. Chronicle Queue is not a message broker in any conventional sense. It is a Java library for inter-process and inter-thread communication that happens to be extraordinarily fast, and it exists because one man — Peter Lawrey — decided that the JVM's standard approach to memory management was an obstacle rather than a feature.

If you are building a web application that processes a few thousand events per second, Chronicle Queue is not for you. Close this chapter and go back to Kafka or RabbitMQ. If you are building a trading system where the difference between 10 microseconds and 100 microseconds is the difference between profit and loss, keep reading. This is your chapter.

---

## Overview

### What It Is

Chronicle Queue is a persisted, low-latency messaging library for Java. It provides an append-only journal (queue) that one or more writer threads can write to and one or more reader threads can read from, with typical latencies measured in single-digit microseconds. Messages are persisted to memory-mapped files and survive process restarts. Multiple processes on the same machine can share a queue through the filesystem.

It is not a network service. There is no broker process, no port to connect to, no cluster to configure. Chronicle Queue is a library that you embed in your Java application. Communication happens through shared access to files on disk. If this sounds primitive, you are not wrong — it is closer to how Unix pipes work than how Kafka works — and that simplicity is precisely why it is so fast.

### Brief History

Chronicle Queue was created by Peter Lawrey, founder of Chronicle Software (originally Higher Frequency Trading Ltd, a name that rather gives away the target market). Lawrey is a figure well known in the Java performance community — the sort of person who has opinions about CPU cache line sizes and knows what `sun.misc.Unsafe` does without consulting the documentation.

The project's origin story is straightforward: financial trading firms needed to pass messages between components of a trading system with minimal, predictable latency. The JVM's standard toolbox — `BlockingQueue`, `ConcurrentLinkedQueue`, NIO channels — was insufficient because:

1. **Garbage collection pauses.** Any solution that allocates objects in the Java heap is subject to GC pauses, which are unpredictable and can range from milliseconds to seconds. In a trading system processing market data, a 50-millisecond GC pause means 50 milliseconds of missed price updates. That is an eternity.

2. **Serialization overhead.** Converting Java objects to byte arrays and back is expensive. Standard serialization frameworks (Java serialization, Kryo, Protobuf) add latency.

3. **No persistence.** In-memory queues are fast but lose data on process restart. Logging to disk typically involves system calls, buffer management, and blocking I/O.

Chronicle Queue addresses all three of these problems through a single mechanism: memory-mapped files accessed off-heap.

The open-source version (Chronicle Queue Community) is available under the Apache License 2.0 with some limitations. Chronicle Queue Enterprise, which adds replication across machines and additional features, is commercially licensed. Chronicle Software operates as a consulting and licensing business, selling to financial institutions and other latency-sensitive organisations.

---

## Architecture

### Memory-Mapped Files

The central idea behind Chronicle Queue is embarrassingly simple: use the operating system's virtual memory system as your message store.

When you create a Chronicle Queue, it creates files on disk (one per "cycle" — typically one file per day, configurable). These files are memory-mapped into the process's address space using `MappedByteBuffer` (or, more precisely, Chronicle's own memory-mapping implementation that bypasses some of the JDK's limitations). Once mapped, reading and writing to the queue is a memory operation — you write bytes to a memory address, and the operating system handles flushing those bytes to disk asynchronously.

This means:

- **No explicit I/O calls.** Writing a message is a memory copy, not a `write()` system call. The OS page cache handles persistence.
- **No serialization to intermediate buffers.** You write directly to the memory-mapped region.
- **No garbage collection impact.** The data lives outside the Java heap, in off-heap memory managed by the OS. The GC does not know about it and does not need to scan it.
- **Persistence is free.** The memory-mapped files are files — they survive process restarts. When you restart your application, you can resume reading from where you left off.

The append-only structure means concurrent writers use a simple sequencing mechanism (a CAS operation on the write position) rather than locks. Readers maintain their own read positions independently and never block writers.

### File Structure and Rolling

Chronicle Queue organises data into rolling files. By default, a new file is created for each day (the "daily" roll cycle). Each file contains a sequence of messages (called "excerpts" in Chronicle terminology), each preceded by a small header containing the message length and metadata.

```
chronicle-queue/
  20260320.cq4       # Today's file
  20260319.cq4       # Yesterday's file
  20260318.cq4       # Day before
  metadata.cq4t      # Index and metadata
```

The `.cq4` files are the actual data files. The `.cq4t` file contains indexing information that allows efficient seeking to specific positions. Old files can be deleted to reclaim disk space — Chronicle Queue supports configurable retention.

Each excerpt in the file has a 4-byte header followed by the message data:

```
[4-byte header: length + metadata flags]
[message bytes]
[4-byte header: length + metadata flags]
[message bytes]
...
```

The header uses specific bit patterns to signal different states: a complete message, a message being written (not yet committed), padding, and end-of-file markers. Readers spin on the header word, waiting for it to transition from "being written" to "complete" — a form of busy-waiting that avoids the overhead of thread parking and notification.

### Lock-Free Design

Chronicle Queue uses no locks for its primary read and write paths. Writers coordinate through compare-and-swap operations on the write position. Readers are completely independent — they simply read from their current position and advance forward.

This lock-free design means:

- **No contention between readers and writers.** A slow reader does not block writers.
- **No contention between multiple readers.** Each reader has its own position.
- **Minimal contention between multiple writers.** The CAS on the write position serialises writes, but each write is a fast memory copy, so contention is brief.

The practical result is that Chronicle Queue scales well with the number of readers and tolerates multiple writers without significant degradation, as long as the machine's memory bandwidth is not saturated.

### Chronicle Wire — The Serialisation Format

Chronicle Wire is the serialisation framework that Chronicle Queue uses by default. It is worth discussing because it is tightly integrated with the queue and contributes significantly to performance.

Wire supports multiple encoding formats:

- **Binary Wire:** A compact binary format optimised for speed. Field names are encoded as numeric codes (determined at compile time or first use), and values are written in their native binary representation. Integers are written as 4 or 8 bytes, not as decimal strings.
- **Text Wire:** A human-readable YAML-like format, useful for debugging and testing.
- **Raw Wire:** No framing at all — just raw bytes. Maximum performance, minimum convenience.
- **JSON Wire:** For interoperability with non-Java systems, though at this point you might question why you are using Chronicle Queue.

The key performance feature of Wire is that it can serialise and deserialise Java objects directly to and from off-heap memory without creating intermediate byte arrays. A `Marshallable` object writes its fields directly to the memory-mapped region. On the read side, fields are read directly from the mapped memory into local variables. No temporary objects are created, no byte arrays are allocated, and the garbage collector remains blissfully unaware that anything happened.

This is what "zero-copy" means in the Chronicle context: the data path from the writer's Java fields to the persistent store on disk involves no copying into intermediate buffers. The fields are written to the memory-mapped address, the OS eventually flushes the page to disk, and the reader reads from the same (or a different mapping of the same) memory-mapped address.

---

## How It Achieves Microsecond Latency

It is worth enumerating specifically why Chronicle Queue is fast, because each design choice contributes to the overall latency profile:

1. **Off-heap storage.** Data never touches the Java heap. The GC never scans it, never moves it, never pauses to collect it. This eliminates the single largest source of latency variability in Java applications.

2. **Memory-mapped I/O.** Writing a message is a memory copy to a mapped region, not a system call. The OS handles persistence asynchronously. There is no `fsync()` on the critical path (by default — you can enable it at the cost of latency).

3. **No serialisation overhead.** Chronicle Wire writes fields directly to memory. There is no intermediate `byte[]` allocation, no serialisation framework overhead, no object allocation.

4. **Lock-free algorithms.** No mutex acquisition, no thread parking, no context switches in the fast path. Writers CAS on the write position. Readers busy-wait on the header word.

5. **Sequential access patterns.** The append-only structure means all writes are sequential, which is optimal for both memory and disk hardware. There is no random I/O, no seeking, no B-tree traversal.

6. **CPU cache friendliness.** Sequential writes and reads keep data in L1/L2 cache. The small header-plus-data format means multiple messages fit in a single cache line.

The combination of these factors produces typical write latencies of 1-2 microseconds for small messages (under a few hundred bytes) and read latencies that are similarly low. The 99th percentile latency is typically within 2-3x of the median, which is remarkable for a JVM-based system. For comparison, Kafka's producer latency is measured in milliseconds, and even Kafka's `acks=0` fire-and-forget mode is orders of magnitude slower.

### Garbage-Free Operation

"Garbage-free" is not marketing hyperbole — it is a specific, measurable claim. In a properly written Chronicle Queue application, the steady-state allocation rate in the Java heap is zero (or near zero). No objects are created during message writing or reading. This means:

- The GC has nothing to collect
- There are no young generation collections interrupting your application
- There are certainly no full GC pauses
- Latency is determined by your code and the hardware, not by the runtime

Achieving true garbage-free operation requires discipline on the application side. If your message handler allocates a `HashMap` for every message, you have re-introduced the problem that Chronicle Queue was designed to avoid. Chronicle provides tooling (`-verbose:gc` analysis, allocation profiling) to help identify and eliminate allocations.

---

## Replication: Chronicle Queue Enterprise

The open-source Chronicle Queue is a single-machine library. If you need data to be replicated to another machine — for disaster recovery, failover, or cross-site distribution — you need Chronicle Queue Enterprise.

Enterprise replication works by:

1. A primary writer appends to a Chronicle Queue on the local filesystem
2. A replication agent reads new excerpts and sends them over the network to one or more replicas
3. Replicas append the received excerpts to their local Chronicle Queue files
4. Consumers on the replica machine read from their local queue

The replication is asynchronous by default, which means there is a small window of data loss if the primary fails before replicated data is acknowledged. Synchronous replication is available but adds network round-trip latency to the write path — defeating the purpose of using Chronicle Queue for many use cases.

Enterprise also adds:

- **Encryption at rest and in transit**
- **Access control** for queue operations
- **Monitoring and metrics** via JMX and Prometheus
- **Time-based and size-based retention management**
- **Delta compression** for replication traffic

The licensing cost is not public and is negotiated per customer. For the target market (financial institutions with more money than patience for open-source support), this is expected. For everyone else, it is a barrier.

---

## Strengths

### Sub-Microsecond Latency

For small messages on modern hardware, Chronicle Queue delivers write latencies under 1 microsecond. This is not a marketing benchmark; it is a measurable property of the library under normal operation. The combination of memory-mapped I/O, off-heap storage, and lock-free algorithms produces latency numbers that are simply unachievable with any network-based message broker.

### Garbage-Free Operation

In a world where JVM garbage collection is the bane of low-latency Java applications, Chronicle Queue's ability to operate without generating garbage is a fundamental advantage. Deterministic latency on the JVM is possible — Chronicle Queue proves it — but it requires staying off-heap.

### Deterministic Performance

The gap between median and tail latency is small. P99 is typically within 2-3x of median. For systems where tail latency matters — and in finance, it always matters — this predictability is as valuable as the raw speed.

### Java Native

If your team writes Java (or Kotlin, or any JVM language), Chronicle Queue integrates naturally. It is a library, not a service. No operational overhead, no cluster management, no network hops. Add a Maven dependency, create a queue, start writing. The learning curve is primarily about understanding the off-heap and garbage-free programming discipline.

### Persistence Without Performance Penalty

Unlike in-memory queues that lose data on restart, Chronicle Queue persists everything to disk via memory mapping — but the write path is a memory operation, not a disk operation. You get persistence without paying for it on the write path. The OS handles flushing to disk asynchronously, and the memory-mapped files survive process crashes (data that has been written to the mapped region is safe even if the process is killed, because it is in the OS page cache).

---

## Weaknesses

### Single-Machine Focus

The open-source version is a single-machine library. There is no built-in networking, no clustering, no distributed anything. If you need to pass messages between machines, you either use Chronicle Queue Enterprise (commercial), build your own network layer on top, or use a different tool. For many modern architectures — microservices running on multiple nodes, cloud-native applications, Kubernetes deployments — this is a fundamental limitation.

### Java Only

Chronicle Queue is a Java library. It is deeply, inextricably Java. The off-heap memory management, the Wire serialisation format, the `Marshallable` interface — these are JVM constructs. If your system includes Python services, Go services, or anything not running on a JVM, Chronicle Queue cannot help you with inter-service communication.

There are some projects that provide non-JVM readers for Chronicle Queue's file format, but they are not first-class, not fully featured, and not what you want to bet a production system on.

### Commercial License for Essential Features

Replication, encryption, access control — features that most production systems need — are behind the Enterprise commercial license. The open-source version is genuinely useful for single-machine inter-thread and inter-process communication, but the moment you need data on more than one machine, you are paying. This is a perfectly reasonable business model, but it limits the addressable use cases for the free version.

### Not a Distributed System

Chronicle Queue does not do leader election, does not do consensus, does not do distributed transactions, does not do partition tolerance. It is a very fast file on a very specific machine. If that machine fails, your queue is unavailable (unless you have Enterprise replication configured). There is no automatic failover, no partition reassignment, no self-healing cluster.

This is not a bug — it is a deliberate design choice. Distributed consensus adds latency, and Chronicle Queue's entire reason for existence is minimising latency. But it means you are responsible for building reliability around it: replication (Enterprise), monitoring, failover procedures, and the knowledge that a single disk failure can take your queue offline.

### Learning Curve for Garbage-Free Programming

Writing garbage-free Java is a skill that most Java developers have never needed to learn. The natural Java idiom — create objects, let the GC clean them up — is exactly what you cannot do in a Chronicle Queue application. This means:

- Object pools instead of `new`
- Primitive fields instead of boxed types
- Pre-allocated buffers instead of dynamic allocation
- Avoiding standard library classes that allocate internally (`String` concatenation, `HashMap`, `ArrayList`)

The tooling is available (allocation profiling, `-verbose:gc` monitoring), but the programming discipline is significant. Teams adopting Chronicle Queue for the first time should budget for a learning curve, especially if they are not already experienced in low-latency Java programming.

---

## Ideal Use Cases

### High-Frequency Trading

This is Chronicle Queue's home turf. Trading systems that need to process market data, calculate signals, and generate orders with deterministic sub-millisecond latency. The pattern is typically: market data comes in from the exchange (via a network interface), is written to a Chronicle Queue, picked up by a strategy component, which writes orders to another Chronicle Queue, which is read by an order gateway and sent to the exchange. Each queue hop adds single-digit microseconds.

### Low-Latency Financial Systems

Beyond trading, any financial system where latency matters: risk calculation engines, position management systems, real-time pricing engines, order management systems. These systems often have the same requirements — fast, deterministic, persistent — and the same JVM ecosystem.

### Inter-Thread and Inter-Process Communication

On a single machine, Chronicle Queue is an excellent inter-process communication mechanism. If you have multiple JVM processes that need to share a message stream — a producer process and multiple consumer processes, for example — Chronicle Queue provides this through the filesystem with better performance and persistence guarantees than named pipes, Unix sockets, or network loopback.

### Audit and Journalling

The append-only, persistent nature of Chronicle Queue makes it a natural fit for audit logging and event journalling. Write every state change, every decision, every external interaction to a Chronicle Queue. The performance overhead is negligible (microseconds per write), the data is durable, and you have a complete, ordered, replayable record of everything that happened.

### When NOT to Use It

- **Distributed systems.** If you need messages to flow between machines as a core requirement, not an afterthought, use a distributed message broker.
- **Multi-language environments.** If your services are written in Python, Go, and Java, Chronicle Queue only serves the Java components.
- **Cloud-native architectures.** Kubernetes pods with ephemeral storage are not a natural fit for memory-mapped files with specific filesystem requirements.
- **General-purpose messaging.** If your latency requirements are measured in milliseconds rather than microseconds, Kafka, NATS, or RabbitMQ are simpler, more flexible, and more broadly supported.

---

## Comparison with Aeron

Chronicle Queue and Aeron (covered in the next chapter) are frequently mentioned together and occasionally confused. They occupy adjacent but distinct niches:

| Aspect | Chronicle Queue | Aeron |
|--------|----------------|-------|
| **Primary use** | Persistent journalling, inter-process messaging | Network messaging, IPC |
| **Transport** | Filesystem (memory-mapped files) | UDP, IPC (shared memory) |
| **Persistence** | Built-in (the queue *is* a file) | Optional (Aeron Archive) |
| **Network support** | Enterprise only | Built-in (UDP unicast/multicast) |
| **Latency focus** | Microsecond writes, GC-free | Nanosecond IPC, microsecond network |
| **Replication** | Enterprise feature | Aeron Cluster (Raft-based) |
| **Philosophy** | Persistent log as the primitive | Transport as the primitive |

The choice between them often comes down to the primary access pattern. If your dominant need is "write an ordered, persistent log that multiple local processes can read," Chronicle Queue is the natural fit. If your dominant need is "send messages between processes with minimal latency, potentially over a network," Aeron is the natural fit.

Many low-latency systems use both: Aeron for network transport between machines, and Chronicle Queue for persistent journalling and local inter-process communication. They are complementary, not competitive.

---

## Code Examples

### Basic Producer and Consumer

```java
import net.openhft.chronicle.queue.ChronicleQueue;
import net.openhft.chronicle.queue.ExcerptAppender;
import net.openhft.chronicle.queue.ExcerptTailer;

public class BasicExample {

    public static void main(String[] args) {
        String queuePath = "/tmp/chronicle-example";

        // Producer
        try (ChronicleQueue queue = ChronicleQueue.single(queuePath)) {
            ExcerptAppender appender = queue.acquireAppender();

            // Write a simple text message
            appender.writeText("Hello, Chronicle Queue");

            // Write structured data using a lambda
            appender.writeDocument(wire -> {
                wire.write("type").text("OrderPlaced");
                wire.write("orderId").text("ord-7829");
                wire.write("amount").float64(149.99);
                wire.write("currency").text("EUR");
                wire.write("timestamp").int64(System.nanoTime());
            });

            System.out.println("Messages written");
        }

        // Consumer
        try (ChronicleQueue queue = ChronicleQueue.single(queuePath)) {
            ExcerptTailer tailer = queue.createTailer();

            // Read the text message
            String text = tailer.readText();
            System.out.println("Read: " + text);

            // Read structured data
            tailer.readDocument(wire -> {
                String type = wire.read("type").text();
                String orderId = wire.read("orderId").text();
                double amount = wire.read("amount").float64();
                String currency = wire.read("currency").text();
                long timestamp = wire.read("timestamp").int64();

                System.out.printf("Order: %s %s %.2f %s at %d%n",
                    type, orderId, amount, currency, timestamp);
            });
        }
    }
}
```

### Using Marshallable Objects (Garbage-Free)

```java
import net.openhft.chronicle.queue.ChronicleQueue;
import net.openhft.chronicle.queue.ExcerptAppender;
import net.openhft.chronicle.queue.ExcerptTailer;
import net.openhft.chronicle.wire.SelfDescribingMarshallable;

public class MarshalExample {

    // Define a message type — no garbage on write or read
    public static class OrderEvent extends SelfDescribingMarshallable {
        private String type;
        private String orderId;
        private double amount;
        private String currency;
        private long timestampNanos;

        // Setters return 'this' for fluent usage
        public OrderEvent type(String type) {
            this.type = type;
            return this;
        }

        public OrderEvent orderId(String orderId) {
            this.orderId = orderId;
            return this;
        }

        public OrderEvent amount(double amount) {
            this.amount = amount;
            return this;
        }

        public OrderEvent currency(String currency) {
            this.currency = currency;
            return this;
        }

        public OrderEvent timestampNanos(long ts) {
            this.timestampNanos = ts;
            return this;
        }

        // Getters
        public String type() { return type; }
        public String orderId() { return orderId; }
        public double amount() { return amount; }
        public String currency() { return currency; }
        public long timestampNanos() { return timestampNanos; }
    }

    public static void main(String[] args) {
        String queuePath = "/tmp/chronicle-marshal-example";

        // Reusable event object — allocated once, reused forever
        OrderEvent event = new OrderEvent();

        try (ChronicleQueue queue = ChronicleQueue.single(queuePath)) {
            ExcerptAppender appender = queue.acquireAppender();

            // Write 1,000,000 messages with zero garbage
            for (int i = 0; i < 1_000_000; i++) {
                event.type("OrderPlaced")
                     .orderId("ord-" + i)   // Note: String concat DOES allocate.
                     .amount(149.99 + i)     // In a truly GC-free system,
                     .currency("EUR")        // you would use a pre-allocated
                     .timestampNanos(System.nanoTime()); // StringBuilder.

                appender.writeDocument(event);
            }

            System.out.println("Wrote 1,000,000 events");
        }

        // Read them back
        OrderEvent readEvent = new OrderEvent(); // Reusable read object

        try (ChronicleQueue queue = ChronicleQueue.single(queuePath)) {
            ExcerptTailer tailer = queue.createTailer();

            int count = 0;
            while (tailer.readDocument(readEvent)) {
                count++;
                if (count % 250_000 == 0) {
                    System.out.printf("Read %d events, latest: %s %s%.2f%n",
                        count, readEvent.type(),
                        readEvent.currency(), readEvent.amount());
                }
            }
            System.out.printf("Total events read: %d%n", count);
        }
    }
}
```

Note the pattern: create a reusable object, populate it for each write, and reuse a read object for each read. This is the garbage-free discipline in action. The `SelfDescribingMarshallable` base class provides efficient serialisation to and from Chronicle Wire format, writing fields directly to the memory-mapped region.

The comment about `String` concatenation is deliberate. True garbage-free programming in Java requires vigilance about every allocation, including implicit ones from string operations, autoboxing, and iterator creation. Most applications will accept a small amount of allocation in non-critical paths and focus garbage-free discipline on the hot path.

### Chronicle Wire Standalone

```java
import net.openhft.chronicle.bytes.Bytes;
import net.openhft.chronicle.wire.Wire;
import net.openhft.chronicle.wire.WireType;

public class WireExample {

    public static void main(String[] args) {
        // Allocate a reusable buffer — off-heap
        Bytes<?> bytes = Bytes.elasticByteBuffer();

        // Write using binary wire (fast, compact)
        Wire wire = WireType.BINARY.apply(bytes);
        wire.write("eventType").text("PriceUpdate");
        wire.write("symbol").text("AAPL");
        wire.write("bid").float64(178.52);
        wire.write("ask").float64(178.55);
        wire.write("timestamp").int64(System.nanoTime());

        System.out.println("Serialized size: " + bytes.readRemaining()
            + " bytes");

        // Read it back
        Wire readWire = WireType.BINARY.apply(bytes);
        String eventType = readWire.read("eventType").text();
        String symbol = readWire.read("symbol").text();
        double bid = readWire.read("bid").float64();
        double ask = readWire.read("ask").float64();
        long ts = readWire.read("timestamp").int64();

        System.out.printf("%s: %s bid=%.2f ask=%.2f%n",
            eventType, symbol, bid, ask);

        // Convert to text wire for debugging
        Bytes<?> textBytes = Bytes.elasticByteBuffer();
        Wire textWire = WireType.TEXT.apply(textBytes);
        textWire.write("eventType").text("PriceUpdate");
        textWire.write("symbol").text("AAPL");
        textWire.write("bid").float64(178.52);
        textWire.write("ask").float64(178.55);

        System.out.println("Text format:\n" + textBytes);

        // Cleanup
        bytes.releaseLast();
        textBytes.releaseLast();
    }
}
```

Chronicle Wire deserves attention because it is the mechanism by which Chronicle Queue avoids the serialisation tax that most messaging systems pay. The binary format is compact (no field name strings in the output, just numeric codes), the encoding is direct (native byte order, no endian conversion for same-architecture communication), and the allocation is zero (off-heap buffers, reused).

---

## Verdict

Chronicle Queue is a precision instrument. It does one thing — low-latency, persistent, local messaging on the JVM — and does it better than anything else available. The sub-microsecond write latency, garbage-free operation, and deterministic performance profile are not theoretical claims but empirically verified properties of the library in real production systems.

The precision comes with constraints. It is Java-only. It is single-machine by default. It is not a distributed system. The enterprise features that most production deployments need — replication, encryption, access control — require a commercial license. The programming discipline required for truly garbage-free operation is non-trivial and unfamiliar to most Java developers.

These constraints are not accidental — they are the direct consequence of the design decisions that make Chronicle Queue fast. Distributed consensus adds latency. Multi-language support adds abstraction layers. GC-friendly programming means objects on the heap. Every feature that Chronicle Queue lacks is a feature whose absence contributes to its performance.

The practical recommendation:

1. **If you are building a low-latency Java system on a single machine** — trading system components, pricing engines, risk calculators, event sourcing journals — Chronicle Queue is likely the best tool available. Nothing else in the JVM ecosystem matches its combination of speed, persistence, and determinism.

2. **If you need inter-machine communication**, evaluate Chronicle Queue Enterprise for replication and consider pairing it with Aeron (next chapter) for network transport. The combination of Chronicle Queue for local persistence and journalling with Aeron for network messaging is common in financial systems.

3. **If your latency requirements are measured in milliseconds**, Chronicle Queue is overkill. Use Kafka, NATS, or RabbitMQ. The operational simplicity and ecosystem breadth of those systems outweigh Chronicle Queue's latency advantage when microseconds do not matter.

4. **If you are not writing Java**, this is not your tool. There is no Python SDK, no Go client, no Node.js binding worth trusting in production.

Chronicle Queue exists because the JVM, despite its many strengths, has a fundamental tension between automatic memory management and deterministic latency. Peter Lawrey's answer was to sidestep the GC entirely, move data off-heap, and treat the filesystem as a communication channel. It is an unorthodox approach that works remarkably well within its constraints. Respect the constraints, and it will reward you with performance that makes other Java developers suspect you are lying about your latency numbers.
