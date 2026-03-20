# Aeron

If Chronicle Queue represents the philosophy that the filesystem is the fastest communication medium on a single machine, Aeron represents a more radical position: that the entire traditional networking stack — TCP, kernel buffers, system calls, context switches — is an unacceptable overhead for systems that need to move data between processes at the speed of hardware. Aeron is what happens when someone with deep knowledge of CPU architecture, operating systems, and network hardware decides that the standard abstractions are the problem.

That someone is Martin Thompson, and his philosophy has a name: mechanical sympathy. The idea, borrowed from racing driver Jackie Stewart's observation that you do not need to be an engineer to drive a car fast but you do need sympathy for the machine, is that the best software performance comes from understanding and respecting the hardware it runs on. CPU cache hierarchies, memory access patterns, branch prediction, kernel bypass — these are not academic concerns for the Aeron community. They are the design parameters.

Aeron is not a message broker. It is a messaging library — a transport layer that moves bytes between processes with extreme efficiency and predictable latency. If you are building systems where nanosecond-level IPC latency and microsecond-level network latency are genuine requirements (not aspirational marketing), Aeron is one of a very small number of tools that can deliver.

---

## Overview

### What It Is

Aeron is a reliable UDP unicast, UDP multicast, and IPC message transport. It provides:

- **Publication and Subscription** abstractions for sending and receiving messages
- **Reliable delivery** over UDP (sequencing, retransmission, flow control)
- **IPC transport** via shared memory for same-machine communication
- **Back-pressure** mechanisms to prevent slow consumers from being overwhelmed
- **Aeron Cluster** for replicated state machines (Raft-based consensus)
- **Aeron Archive** for persistent message recording and replay

Aeron operates at a lower level than systems like Kafka or RabbitMQ. There are no topics (in the Kafka sense), no queues, no routing rules, no broker process deciding where messages go. A publisher sends messages on a channel (identified by a URI-like address) and stream ID, and subscribers listening on the same channel and stream receive them. That is the model. Everything above this — topic semantics, consumer groups, message routing, persistence — is your responsibility to build if you need it.

### Brief History

Aeron was created by Martin Thompson and Todd Montgomery at Real Logic, with the first public release around 2014. Thompson had previously worked on the LMAX Disruptor (the inter-thread messaging library that demonstrated that lock-free ring buffers could achieve millions of messages per second on a single thread) and spent years consulting on low-latency systems for financial services.

The motivation for Aeron was dissatisfaction with existing messaging transports. TCP, for all its reliability, adds latency through congestion control algorithms (Nagle's algorithm, slow start), kernel buffer copies, and system call overhead. UDP avoids some of these costs but provides no reliability — messages can be lost, duplicated, or reordered. Existing reliable UDP libraries were either unmaintained, poorly designed, or encumbered by unsuitable licenses.

Aeron's design goals were explicit:

1. **Predictable, low latency.** Not just low mean latency — low *tail* latency. The P99.9 should be close to the median.
2. **High throughput.** Millions of small messages per second between processes.
3. **Reliable delivery.** Guaranteed, ordered delivery over an unreliable transport (UDP).
4. **Zero-allocation steady state.** No garbage collection pressure during message sending or receiving.
5. **Mechanical sympathy.** Design every data structure and algorithm to work with modern hardware, not against it.

Aeron is open source under the Apache License 2.0. Real Logic provides commercial support and consulting. The codebase is available in Java (the primary implementation), C (a native implementation for non-JVM environments), and C++ (wrapping the C implementation). The Java and C implementations are maintained in parallel and interoperable — a Java publisher can send to a C subscriber and vice versa.

Thompson continues to lead development. The community, while small compared to Kafka or RabbitMQ, includes some of the most knowledgeable low-latency systems engineers in the industry. The project's GitHub issues and mailing list discussions read like a graduate seminar in systems programming. This is both an asset (high-quality support) and an honest assessment of the target audience.

---

## Architecture

### The Media Driver

The central component of Aeron's architecture is the **Media Driver**. The media driver is the process (or thread) responsible for all I/O: sending UDP packets, receiving UDP packets, managing IPC shared memory regions, handling retransmissions, and performing flow control.

The media driver can run in three modes:

- **Dedicated:** A separate process with its own JVM (or native process for the C driver). Client applications communicate with the driver through shared memory.
- **Embedded:** Running as threads within the client application's process. Lower latency (no inter-process communication with the driver) but ties the driver's lifecycle to the application.
- **Shared:** A single thread handles both sending and receiving. Lower resource usage but potentially higher latency under load.

For lowest latency, the dedicated or embedded driver with a dedicated polling thread is preferred. The driver thread typically busy-waits (spinning on the CPU) rather than blocking, which means it consumes a full CPU core but eliminates the latency of thread wake-up.

```
┌─────────────────────────────────────────────────────────┐
│                     Application                          │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │Publisher  │  │ Subscriber   │  │ Subscriber   │      │
│  └─────┬────┘  └──────┬───────┘  └──────┬───────┘      │
│        │               │                  │              │
│        ▼               ▼                  ▼              │
│  ┌─────────────────────────────────────────────────┐    │
│  │           Shared Memory (CnC file)              │    │
│  └─────────────────────┬───────────────────────────┘    │
└────────────────────────┼────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    Media Driver                          │
│  ┌──────────┐  ┌──────────────┐  ┌─────────────┐       │
│  │ Sender   │  │  Receiver    │  │ Conductor   │       │
│  └─────┬────┘  └──────┬───────┘  └─────────────┘       │
│        │               │                                 │
│        ▼               ▼                                 │
│    UDP/IPC         UDP/IPC                               │
└─────────────────────────────────────────────────────────┘
```

The Command and Control (CnC) file is a memory-mapped file shared between the client application and the media driver. It contains ring buffers for commands (from client to driver) and for received messages (from driver to client). This shared-memory architecture means that sending a message from the application to the driver is a memory write, not a system call — the same principle that makes Chronicle Queue fast, applied to the control plane.

### Publications and Subscriptions

The API model is straightforward:

- A **Publication** is a handle for sending messages. It is associated with a channel (transport address) and a stream ID (logical message stream within a channel).
- A **Subscription** is a handle for receiving messages. It subscribes to a channel and one or more stream IDs.
- Messages are sent as byte buffers. Aeron does not care about message format — it transports bytes.

```
Publication  → Channel: aeron:udp?endpoint=224.0.1.1:40456  Stream: 1001
Subscription → Channel: aeron:udp?endpoint=224.0.1.1:40456  Stream: 1001
```

The channel URI specifies the transport:

- `aeron:udp?endpoint=host:port` — UDP unicast to a specific host
- `aeron:udp?endpoint=224.0.1.1:40456|interface=eth0` — UDP multicast
- `aeron:ipc` — shared memory IPC (same machine only)

A single media driver can manage multiple publications and subscriptions across different channels and transports simultaneously.

### Reliable UDP

Aeron implements reliability over UDP through:

- **Sequencing.** Each message has a position (a monotonically increasing byte offset within the stream). Subscribers track their position and detect gaps.
- **NAK-based retransmission.** When a subscriber detects a gap in the sequence, it sends a NAK (negative acknowledgement) to the publisher, which retransmits the missing data. This is more efficient than ACK-based schemes because it generates no traffic when everything is working correctly.
- **Flow control.** Publishers are prevented from sending faster than subscribers can consume. The flow control strategy is pluggable; the default uses a window-based approach similar to TCP's sliding window but with fewer round trips.
- **Heartbeats.** Publishers and subscribers send periodic heartbeats to detect connection loss.

This reliability layer provides ordered, lossless delivery over UDP — the reliability of TCP without TCP's overhead. The key differences from TCP:

- No Nagle's algorithm (no buffering small messages waiting for a full packet)
- No slow start (no gradual ramp-up of sending rate)
- No head-of-line blocking (a lost packet only blocks messages in that specific stream)
- No kernel-level buffer management (data goes directly from application memory to the NIC, where possible)

### IPC Transport

For same-machine communication, Aeron's IPC transport bypasses the network stack entirely. Publisher and subscriber share a memory-mapped log file. Writing a message is a memory copy into the shared region; reading a message is a memory read from the shared region.

IPC latency is measured in nanoseconds — typically 50-200 nanoseconds for small messages on modern hardware. This is faster than a `localhost` UDP loopback by orders of magnitude, because there is no kernel involvement, no socket buffer copy, no interrupt handling.

The IPC transport is particularly useful for:

- Communication between components in a trading system (market data handler, strategy engine, order gateway)
- High-performance pipelines where stages run in separate processes for isolation but need minimal communication overhead
- Any scenario where you would use shared memory but want a clean API with flow control and back-pressure

---

## Aeron Cluster

Aeron Cluster extends Aeron with **replicated state machine** semantics based on the Raft consensus protocol. This is Aeron's answer to the question: "How do I build a fault-tolerant system with Aeron?"

A cluster consists of:

- **Multiple nodes** (typically 3 or 5) each running a copy of your state machine
- **A leader** node that processes client requests and replicates them to followers
- **Followers** that maintain copies of the replicated log and can take over if the leader fails
- **Client sessions** that connect to the cluster and send commands

The programming model is an event-sourcing style: clients send commands to the cluster, the leader sequences them into a replicated log, and all nodes apply the commands to their state machines in the same order. This guarantees that all nodes have identical state.

Aeron Cluster is not a message broker. It is a framework for building fault-tolerant services. If you want to build a matching engine, a sequencer, or an order management system that survives node failures without losing state, Aeron Cluster provides the replication and leader election infrastructure. You provide the state machine logic.

The latency characteristics are impressive: cluster commit latency (the time from a client sending a command to receiving confirmation that it is replicated to a majority) is typically in the low hundreds of microseconds over a LAN. This is orders of magnitude faster than Kafka's replication or any database commit.

---

## Aeron Archive

Aeron Archive adds **persistent recording and replay** to Aeron streams. Without Archive, Aeron is an ephemeral transport — messages exist only while they are in the log buffers. Archive records stream data to disk and provides APIs for replaying recorded data.

Use cases for Archive include:

- **Audit logging.** Record all messages for regulatory compliance or debugging.
- **Late joiners.** A new subscriber can replay historical data to build its initial state.
- **Replay-based recovery.** After a crash, replay recorded messages to reconstruct state.
- **Time-travel debugging.** Replay a specific time range to reproduce a production issue.

Archive records are stored as files on the local filesystem. They can be replicated to other machines using Aeron's standard transport.

The combination of Aeron transport (live messaging) + Aeron Archive (persistence) + Aeron Cluster (replication) provides a complete platform for building fault-tolerant, low-latency distributed systems. It is not a turnkey solution — assembling these components requires significant engineering effort — but the building blocks are sound.

---

## Simple Binary Encoding (SBE)

SBE is not technically part of Aeron, but it is Martin Thompson's companion project and is used extensively alongside Aeron. SBE is a serialisation format designed for the same constraints as Aeron: zero allocation, minimal CPU overhead, and direct memory access.

SBE works by:

1. You define a message schema in XML
2. The SBE compiler generates Java (or C, C++) codec classes
3. The codec reads and writes fields directly from/to a byte buffer — no intermediate object allocation
4. Fields are at fixed offsets within the buffer, so reading field N does not require parsing fields 1 through N-1

```xml
<!-- SBE schema for a market data message -->
<sbe:messageSchema package="com.example.sbe"
                    id="1" version="0"
                    semanticVersion="1.0"
                    byteOrder="littleEndian">

    <types>
        <type name="Symbol" primitiveType="char" length="8"/>
    </types>

    <sbe:message name="PriceUpdate" id="1">
        <field name="symbol" id="1" type="Symbol"/>
        <field name="bidPrice" id="2" type="int64"/>
        <field name="askPrice" id="3" type="int64"/>
        <field name="bidSize" id="4" type="int32"/>
        <field name="askSize" id="5" type="int32"/>
        <field name="timestampNanos" id="6" type="int64"/>
    </sbe:message>
</sbe:messageSchema>
```

The generated codec provides direct access methods:

```java
// Writing (zero allocation)
priceUpdateEncoder.wrap(buffer, offset)
    .symbol("AAPL    ")
    .bidPrice(17852)    // Price in hundredths of a cent
    .askPrice(17855)
    .bidSize(500)
    .askSize(300)
    .timestampNanos(System.nanoTime());

// Reading (zero allocation)
priceUpdateDecoder.wrap(buffer, offset,
    PriceUpdateDecoder.BLOCK_LENGTH,
    PriceUpdateDecoder.SCHEMA_VERSION);

String symbol = priceUpdateDecoder.symbol();  // Direct read from buffer
long bid = priceUpdateDecoder.bidPrice();      // No parsing, no allocation
long ask = priceUpdateDecoder.askPrice();
```

SBE messages are typically 10-100x smaller than JSON and 2-5x smaller than Protobuf for the same content. Encoding and decoding times are measured in nanoseconds. The trade-off is rigidity: SBE messages have fixed schemas, field reordering requires schema changes, and variable-length data (strings, arrays) is more cumbersome than in Protobuf or JSON.

For Aeron-based systems, SBE is the natural serialisation choice. The combination of Aeron's zero-allocation transport and SBE's zero-allocation serialisation means the entire message path — from application fields to network packet — involves no heap allocation.

---

## Strengths

### Predictable Nanosecond-Level Latency

Aeron's IPC transport delivers sub-microsecond latency. Network transport over UDP delivers single-digit microsecond latency on a LAN. These are not benchmarks measured on idle systems — they are the operational profile under load. The P99 is close to the median because there are no GC pauses, no lock contention, no kernel buffer copies adding sporadic latency spikes.

### Zero-Allocation Design

Like Chronicle Queue, Aeron operates without generating garbage in steady state. The media driver, publications, subscriptions, and message handling all operate without heap allocation. This is fundamental to achieving predictable latency on the JVM.

### Mechanical Sympathy

Every data structure in Aeron is designed for the hardware it runs on:

- **Ring buffers** use padding to prevent false sharing between producer and consumer cache lines
- **Log buffers** are sized to align with OS page sizes
- **Counters** use memory-mapped files for zero-copy sharing between driver and client
- **Busy-wait loops** keep threads on-CPU, avoiding the latency of context switches

This is not premature optimisation — it is the necessary foundation for nanosecond-level performance. At these latencies, a single cache miss (100+ nanoseconds) or context switch (1-10 microseconds) is a significant portion of the total time budget.

### IPC Transport

Aeron's IPC transport is arguably its most impressive feature. Shared-memory communication with flow control, back-pressure, and a clean API. It is what Unix shared memory should have been: fast, safe, and usable without deep kernel knowledge.

### Multi-Language Support

Unlike Chronicle Queue (Java only), Aeron has production-quality implementations in Java and C/C++. A Java publisher can send to a C subscriber. This is critical for systems that span languages — a Java strategy engine sending orders to a C++ gateway, for example.

### The Building Blocks Approach

Aeron, Aeron Cluster, Aeron Archive, and SBE form a coherent set of building blocks for low-latency distributed systems. They are designed to work together but are independently useful. You can use Aeron transport without Cluster. You can use SBE without Aeron. This modularity lets you adopt what you need without buying into an all-or-nothing framework.

---

## Weaknesses

### Steep Learning Curve

Aeron is not a system you pick up in an afternoon. The concepts — media drivers, channels, stream IDs, fragment handlers, back-pressure, log buffers — are unfamiliar to most developers. The documentation is accurate but assumes systems programming knowledge. The error messages are descriptive but assume you understand why a publication might be "back-pressured" or why a subscription's "image" might be "unavailable."

The community is helpful but small. There is no "Aeron for Beginners" ecosystem of blog posts and video tutorials. The canonical learning resources are Martin Thompson's conference talks, the project's wiki, and reading the source code. If you are comfortable with that, you will be fine. If you need a gentle on-ramp, budget significant time for learning.

### Not a General-Purpose Broker

Aeron does not have topics, queues, routing rules, consumer groups, dead-letter handling, message filtering, or any of the features that general-purpose brokers provide. It is a transport layer. If you need broker semantics, you build them on top of Aeron or use a different tool.

This is a deliberate design choice — broker features add latency and complexity — but it means that using Aeron for anything beyond point-to-point or multicast messaging requires significant application-level development.

### Java/C++/C Only

Three languages is more than Chronicle Queue's one, but it is still a limitation. If your system includes Python, Go, Rust, or .NET services, those services cannot use Aeron directly. There are community bindings for some languages (Rust and .NET notably), but they vary in completeness and maintenance status. The primary implementations are Java and C.

### Requires Deep Systems Knowledge

Running Aeron well — especially for the lowest latency — requires understanding of:

- CPU affinity and pinning (keeping the media driver thread on a specific core)
- NUMA topology (ensuring the driver thread and memory are on the same NUMA node)
- Network interface configuration (interrupt coalescing, ring buffer sizes)
- OS tuning (huge pages, scheduler settings, network stack parameters)
- JVM tuning (GC configuration, JIT compilation, safepoints)

A misconfigured Aeron deployment can perform worse than a well-configured TCP solution. The defaults are reasonable for development, but production deployments targeting the lowest latency require expert tuning.

### Operational Complexity

There is no Aeron web UI, no Grafana dashboard out of the box, no management CLI with friendly output. Aeron exposes counters through memory-mapped files, which can be read by monitoring tools, but the monitoring infrastructure is your responsibility to build. The `AeronStat` tool provides counter values, but interpreting them requires understanding of Aeron's internals.

For teams accustomed to Kafka's JMX metrics, Prometheus exporters, and third-party monitoring dashboards, Aeron's operational tooling feels sparse. You are expected to know what you are doing.

---

## Ideal Use Cases

### Trading Systems

Aeron's natural habitat. Market data distribution, order routing, position updates, risk calculations — any component of a trading system that needs to move data between processes with minimal, predictable latency. The IPC transport for intra-machine communication and UDP multicast for market data distribution are purpose-built for this domain.

### Real-Time Pricing

Systems that calculate and distribute prices in real time — foreign exchange rates, options pricing, bond yields — where stale prices have direct financial consequences. Aeron's multicast transport is particularly suitable: one publisher, many subscribers, all receiving the same data simultaneously.

### Systems Where GC Pauses Are Unacceptable

Any JVM-based system where a 10-millisecond GC pause causes a measurable business impact. This goes beyond trading to include real-time control systems, live audio/video processing, and interactive gaming servers.

### High-Performance Microservices Communication

For microservices architectures where inter-service latency is a critical constraint, Aeron's IPC transport (for co-located services) and UDP transport (for distributed services) offer dramatically lower latency than HTTP/gRPC or even most message brokers. The trade-off is operational complexity and the need to build service discovery, load balancing, and routing logic yourself.

---

## Operational Reality

### Media Driver Tuning

The media driver is the heart of Aeron's performance, and tuning it is the most impactful operational task:

- **Thread mode.** Dedicated threads (one for sending, one for receiving, one for the conductor) provide the best performance. Shared mode (one thread for everything) reduces CPU usage but increases latency.
- **Busy-wait vs. back-off.** Busy-wait (spinning) provides the lowest latency but consumes a full CPU core per thread. Back-off strategies (yielding, sleeping) reduce CPU usage at the cost of latency.
- **Buffer sizes.** Publication and subscription log buffer sizes affect throughput and memory usage. Larger buffers tolerate more burst traffic but consume more memory.
- **Term length.** The log buffer term length affects how much data can be in-flight. Default is 64KB, which is suitable for most workloads.

### CPU Pinning

For lowest latency, the media driver threads should be pinned to specific CPU cores using `taskset` or `isolcpus`. This prevents the OS scheduler from migrating threads between cores, which would cause cache invalidation and latency spikes.

```bash
# Pin the media driver to cores 2 and 3
taskset -c 2,3 java -cp aeron-all.jar \
    io.aeron.driver.MediaDriver \
    aeron.threading.mode=DEDICATED \
    aeron.sender.idle.strategy=noop \
    aeron.receiver.idle.strategy=noop
```

On NUMA systems, ensure the pinned cores and the memory used by the driver are on the same NUMA node. Cross-NUMA memory access adds 50-100 nanoseconds per access — measurable at Aeron's latency scale.

### DPDK Considerations

For the absolute lowest network latency, some Aeron deployments use DPDK (Data Plane Development Kit) to bypass the kernel's network stack entirely. DPDK provides user-space network drivers that read and write packets directly from/to the NIC's memory, eliminating kernel overhead.

Aeron does not include DPDK integration out of the box, but the C media driver can be modified to use DPDK for packet I/O. This is deep systems work — you are essentially taking ownership of the network interface from the OS — but it can reduce network latency from single-digit microseconds to hundreds of nanoseconds.

Whether DPDK is worth the complexity depends on your latency requirements and your team's capability. For most Aeron deployments, standard UDP with tuned kernel settings is sufficient. DPDK is for the last few microseconds, and extracting those microseconds requires expertise that is expensive and rare.

---

## Aeron vs Chronicle Queue vs Kernel Bypass

These three technologies are frequently mentioned together and occasionally conflated. Here is how they compare:

| Aspect | Aeron | Chronicle Queue | Kernel Bypass (DPDK/RDMA) |
|--------|-------|----------------|--------------------------|
| **Primary function** | Message transport | Persistent journal | Raw packet I/O |
| **Network support** | UDP unicast/multicast | Enterprise only | Direct NIC access |
| **IPC** | Shared memory | Memory-mapped files | N/A (network only) |
| **Persistence** | Archive (optional) | Built-in | None |
| **Reliability** | Built-in (NAK-based) | N/A (local only) | Application's problem |
| **Latency (IPC)** | 50-200 ns | 1-2 us | N/A |
| **Latency (network)** | 2-10 us | N/A (no networking) | 0.5-2 us |
| **Languages** | Java, C, C++ | Java | C (primarily) |
| **Abstraction level** | Transport | Storage | Hardware |
| **Operational complexity** | High | Medium | Very high |

The relationship between them:

- **Aeron** is a transport. It moves bytes between processes efficiently. It does not store them long-term (without Archive).
- **Chronicle Queue** is a store. It persists ordered messages to disk efficiently. It does not move them between machines (without Enterprise).
- **Kernel bypass** is infrastructure. It gives you raw access to network hardware. It provides no messaging semantics at all.

A complete low-latency system might use all three: kernel bypass (DPDK) for receiving raw market data from an exchange, Aeron for distributing that data between internal components, and Chronicle Queue for journalling every message for audit and replay. Each tool handles the layer it is designed for.

Alternatively, many systems use Aeron alone (its built-in UDP handling is sufficient for most purposes) with Chronicle Queue for persistence. Adding kernel bypass is a significant engineering investment that is justified only when Aeron's standard UDP latency is insufficient — which is a rare requirement outside of the most competitive trading environments.

---

## Code Examples

### Basic Publication and Subscription

```java
import io.aeron.Aeron;
import io.aeron.Publication;
import io.aeron.Subscription;
import io.aeron.driver.MediaDriver;
import io.aeron.logbuffer.FragmentHandler;
import org.agrona.BufferUtil;
import org.agrona.concurrent.UnsafeBuffer;

public class AeronBasicExample {

    private static final String CHANNEL = "aeron:udp?endpoint=localhost:40123";
    private static final int STREAM_ID = 1001;

    public static void main(String[] args) throws Exception {
        // Start an embedded media driver
        try (MediaDriver driver = MediaDriver.launchEmbedded();
             Aeron aeron = Aeron.connect(
                 new Aeron.Context().aeronDirectoryName(
                     driver.aeronDirectoryName()))) {

            // Create a publication (sender)
            try (Publication publication = aeron.addPublication(
                     CHANNEL, STREAM_ID)) {

                // Create a subscription (receiver)
                try (Subscription subscription = aeron.addSubscription(
                         CHANNEL, STREAM_ID)) {

                    // Wait for the subscription to be connected
                    while (!subscription.isConnected()) {
                        Thread.yield();
                    }

                    // Prepare a message buffer (reusable, no allocation
                    // in the send loop)
                    UnsafeBuffer buffer = new UnsafeBuffer(
                        BufferUtil.allocateDirectAligned(256, 64));

                    // Send messages
                    for (int i = 0; i < 10; i++) {
                        String message = "Order-" + i;
                        buffer.putStringWithoutLengthAscii(0, message);

                        // Offer the message to the publication
                        long result;
                        while ((result = publication.offer(
                                buffer, 0, message.length())) < 0) {
                            // Back-pressured or not connected — retry
                            if (result == Publication.BACK_PRESSURED) {
                                Thread.yield();
                            } else if (result == Publication.NOT_CONNECTED) {
                                Thread.sleep(1);
                            }
                        }
                        System.out.println("Sent: " + message);
                    }

                    // Receive messages
                    FragmentHandler handler = (directBuffer, offset,
                            length, header) -> {
                        byte[] data = new byte[length];
                        directBuffer.getBytes(offset, data);
                        System.out.println("Received: "
                            + new String(data));
                    };

                    int received = 0;
                    while (received < 10) {
                        int fragments = subscription.poll(handler, 10);
                        received += fragments;
                        if (fragments == 0) {
                            Thread.yield();
                        }
                    }
                }
            }
        }
    }
}
```

Note the explicit handling of back-pressure on the publication side. When `offer()` returns a negative value, the publisher must decide what to do: retry, yield, drop the message, or apply application-level back-pressure. This is a fundamental difference from broker-based systems where the broker absorbs back-pressure. In Aeron, back-pressure is the publisher's problem, and ignoring it is a bug.

Also note the `FragmentHandler` callback pattern. Aeron delivers messages as fragments — a message may span multiple fragments if it exceeds the MTU. The standard `FragmentAssembler` handles reassembly, but for maximum performance, designing messages to fit within a single fragment avoids the reassembly overhead.

### IPC Example (Shared Memory)

```java
import io.aeron.Aeron;
import io.aeron.Publication;
import io.aeron.Subscription;
import io.aeron.driver.MediaDriver;
import io.aeron.driver.ThreadingMode;
import io.aeron.logbuffer.FragmentHandler;
import org.agrona.BufferUtil;
import org.agrona.concurrent.BusySpinIdleStrategy;
import org.agrona.concurrent.IdleStrategy;
import org.agrona.concurrent.UnsafeBuffer;

public class AeronIpcExample {

    // IPC channel — no network, shared memory only
    private static final String CHANNEL = "aeron:ipc";
    private static final int STREAM_ID = 2001;

    public static void main(String[] args) throws Exception {
        // Configure driver for lowest latency
        MediaDriver.Context driverCtx = new MediaDriver.Context()
            .threadingMode(ThreadingMode.DEDICATED)
            .conductorIdleStrategy(new BusySpinIdleStrategy())
            .senderIdleStrategy(new BusySpinIdleStrategy())
            .receiverIdleStrategy(new BusySpinIdleStrategy());

        try (MediaDriver driver = MediaDriver.launch(driverCtx);
             Aeron aeron = Aeron.connect(
                 new Aeron.Context().aeronDirectoryName(
                     driver.aeronDirectoryName()))) {

            Publication publication = aeron.addPublication(
                CHANNEL, STREAM_ID);
            Subscription subscription = aeron.addSubscription(
                CHANNEL, STREAM_ID);

            // Wait for connection
            while (!subscription.isConnected()) {
                Thread.yield();
            }

            UnsafeBuffer buffer = new UnsafeBuffer(
                BufferUtil.allocateDirectAligned(64, 64));

            // Publisher thread
            Thread publisher = new Thread(() -> {
                IdleStrategy idle = new BusySpinIdleStrategy();
                for (int i = 0; i < 1_000_000; i++) {
                    buffer.putLong(0, System.nanoTime());
                    buffer.putInt(8, i);

                    while (publication.offer(buffer, 0, 12) < 0) {
                        idle.idle();
                    }
                }
            }, "publisher");

            // Subscriber thread — measure latency
            long[] latencies = new long[1_000_000];
            Thread subscriber = new Thread(() -> {
                IdleStrategy idle = new BusySpinIdleStrategy();
                int[] count = {0};

                FragmentHandler handler = (buf, offset, length, header) -> {
                    long sendTime = buf.getLong(offset);
                    long latency = System.nanoTime() - sendTime;
                    if (count[0] < latencies.length) {
                        latencies[count[0]++] = latency;
                    }
                };

                while (count[0] < 1_000_000) {
                    int fragments = subscription.poll(handler, 10);
                    if (fragments == 0) {
                        idle.idle();
                    }
                }
            }, "subscriber");

            subscriber.start();
            publisher.start();

            publisher.join();
            subscriber.join();

            // Report latency statistics
            java.util.Arrays.sort(latencies);
            System.out.printf("IPC Latency (nanoseconds):%n");
            System.out.printf("  Median:  %,d ns%n",
                latencies[500_000]);
            System.out.printf("  P99:     %,d ns%n",
                latencies[990_000]);
            System.out.printf("  P99.9:   %,d ns%n",
                latencies[999_000]);
            System.out.printf("  P99.99:  %,d ns%n",
                latencies[999_900]);
            System.out.printf("  Max:     %,d ns%n",
                latencies[999_999]);

            publication.close();
            subscription.close();
        }
    }
}
```

This example measures IPC latency end-to-end. On a modern server with CPU pinning configured, you can expect median latencies around 50-200 nanoseconds and P99 under 1 microsecond. These numbers sound implausible until you realise that the message path is: write 12 bytes to a memory-mapped region (the publication log buffer), the media driver copies them to the subscription log buffer (another memory-mapped region), and the subscriber reads them. There are no system calls, no kernel transitions, no network stack involvement.

The `BusySpinIdleStrategy` on all threads means every thread consumes a full CPU core at 100% utilisation. This is the latency-optimal configuration and the resource-expensive one. For systems where CPU cores are less abundant than latency budgets, the `BackoffIdleStrategy` provides a configurable spin-then-yield-then-sleep sequence.

---

## Verdict

Aeron is the most technically impressive messaging technology covered in this book. The combination of reliable UDP transport, nanosecond-level IPC, zero-allocation design, and mechanical sympathy produces a system that operates at the boundary of what software can achieve on commodity hardware. Martin Thompson and the Real Logic team have built something that genuinely pushes the state of the art in messaging performance.

It is also the least accessible. The learning curve is steep. The operational requirements are demanding. The ecosystem is minimal. The documentation assumes expertise. Building a complete system on Aeron requires significantly more engineering effort than using Kafka or RabbitMQ because Aeron provides the transport layer and leaves everything above it — routing, persistence, consumer management, monitoring — to you.

This is not a criticism. It is a statement of what Aeron is: a high-performance building block for teams that know exactly what they need and have the expertise to build it. Aeron does not try to be everything to everyone. It tries to be the fastest message transport available, and it succeeds.

The practical recommendation:

1. **If you are building a low-latency trading system** or similar performance-critical infrastructure, Aeron should be on your shortlist. Evaluate it alongside kernel bypass solutions and commercial offerings from LMAX, Solace, and others. Aeron's open-source availability and clean design make it a strong foundation.

2. **If you need IPC between co-located processes** and are willing to accept the operational overhead, Aeron's IPC transport is unmatched. Nothing else provides sub-microsecond inter-process communication with flow control and back-pressure through a usable API.

3. **If your latency requirements are measured in milliseconds**, Aeron is the wrong tool. It is designed for microsecond and nanosecond latencies. Using it for a system where 10-millisecond latency is acceptable means paying the complexity cost without realising the performance benefit. Use NATS, Kafka, or even HTTP.

4. **If your team does not include systems-level engineers** who understand CPU affinity, NUMA topology, and memory-mapped I/O, be honest about the operational investment. Aeron in the hands of an experienced low-latency team is transformative. Aeron in the hands of a team that primarily writes Spring Boot applications is a liability.

5. **Consider the combination.** Aeron for transport, Chronicle Queue for persistence, SBE for serialisation. These tools were designed by people who talk to each other, share a philosophy, and built complementary systems. The whole is greater than the sum of the parts — if you need the whole.

Aeron exists because Martin Thompson believed that messaging could be faster than it was, and he was right. The question is not whether Aeron is impressive — it is whether your problem requires that level of performance, and whether your team can pay the engineering cost to wield it. For the right problem and the right team, there is nothing else quite like it.
