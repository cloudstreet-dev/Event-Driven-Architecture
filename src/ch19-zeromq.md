# ZeroMQ

Every other chapter in this section describes a broker — a server process that sits between producers and consumers, accepting messages, storing them, and routing them to the right place. ZeroMQ is not that. ZeroMQ is a library. There is no server. There is no broker. There is no daemon to install, configure, monitor, or page you about at 3 AM. This is simultaneously its greatest strength and the thing that confuses people most about it.

ZeroMQ describes itself as "sockets on steroids," which is the kind of tagline that either intrigues you or makes you nervous, depending on your relationship with socket programming. The more precise description is: ZeroMQ is a high-performance asynchronous messaging library that gives you smart sockets with built-in patterns for common distributed computing problems. It handles the transport (TCP, IPC, inproc, multicast), the framing, the reconnection, the buffering, and the routing. You handle everything else.

That "everything else" is the part they put in smaller font.

---

## Overview

### Philosophy

ZeroMQ's design philosophy is radical in the messaging world: the network *is* the broker. Instead of routing messages through a central server, ZeroMQ embeds the messaging logic directly in your application. Each endpoint is both a sender and a receiver, and the library handles the messy details of connection management, message framing, and I/O multiplexing.

This is not a new idea — it is how BSD sockets work, conceptually. What ZeroMQ adds on top of raw sockets is intelligence: automatic reconnection, message queuing, fan-out patterns, load balancing, and a clean API that abstracts away the worst of systems-level network programming. It is the difference between hand-rolling HTTP on top of TCP and using a library that handles the protocol for you — except ZeroMQ gives you building blocks rather than a finished protocol.

The design principles are:
1. **No broker.** The fastest message is the one that does not pass through an intermediary.
2. **Smart endpoints, dumb network.** The complexity lives in the application, not in infrastructure.
3. **Patterns, not protocols.** ZeroMQ provides socket *types* that encode messaging patterns (request-reply, publish-subscribe, etc.) rather than implementing a specific application protocol.
4. **Zero-copy where possible.** Performance is a first-order design concern, not an afterthought.

### History

ZeroMQ was created by iMatix Corporation, primarily by Pieter Hintjens and Martin Sustrik. Hintjens was one of those rare figures in open source — a brilliant programmer, a gifted writer, and an absolute force of nature in community building. His book *ZeroMQ: Messaging for Many Applications* (commonly known as "the zguide") is not just the best ZeroMQ documentation; it is one of the best pieces of technical writing in the distributed systems space. It is also free to read online.

The project started around 2007, with the first stable release in 2010. The core library (libzmq) is written in C++, which contributes to its performance characteristics and its ability to provide bindings for virtually every programming language in existence.

Hintjens passed away in 2016, but the project continues under community stewardship. The ZeroMQ RFC process (based on the Collective Code Construction Contract, or C4, which Hintjens created) is itself noteworthy as a model for open-source governance.

Martin Sustrik, the other co-creator, left the ZeroMQ project and went on to create nanomsg (and later, nng) — spiritual successors that we will cover briefly at the end of this chapter. The split was philosophical and somewhat acrimonious, which is the natural state of open-source projects started by people with strong opinions.

---

## Architecture

### Socket Types

ZeroMQ's architecture is defined by its socket types. Each socket type encodes a specific messaging pattern, and the combinations of socket types create the distributed computing patterns you build with.

**REQ/REP (Request-Reply)**

The simplest pattern. A `REQ` socket sends a request and then blocks until it receives a reply. A `REP` socket receives a request and then sends a reply. Strictly synchronous, strictly alternating. The socket types enforce the send-receive-send-receive cadence — if you try to send two messages in a row on a `REQ` socket, ZeroMQ will complain.

```
REQ ──── request ────> REP
REQ <──── reply ────── REP
```

Useful for RPC-style communication. Fragile in practice because if either side dies mid-exchange, the surviving socket enters a confused state. This is why you rarely use raw REQ/REP in production — you use DEALER/ROUTER instead.

**PUB/SUB (Publish-Subscribe)**

A `PUB` socket sends messages to all connected `SUB` sockets. `SUB` sockets subscribe to specific message prefixes (or receive everything). Messages flow one way only — `PUB` to `SUB`.

```
PUB ──── msg ────> SUB (subscribed to "orders.")
    ──── msg ────> SUB (subscribed to "")  // all messages
    ──── msg ────> SUB (subscribed to "payments.")
```

The subscription filtering happens at the publisher side (in recent versions), which means unmatched messages are not sent over the wire. This is important for performance but means the publisher bears the CPU cost of filtering.

A critical subtlety: PUB/SUB has no delivery guarantees. If a subscriber is slow, messages are dropped (after the high-water mark is reached). If a subscriber connects after a message was sent, that message is gone. There is no persistence, no replay, no acknowledgment. This is by design.

**PUSH/PULL (Pipeline)**

`PUSH` sends messages to connected `PULL` sockets using round-robin load balancing. Messages flow one way. This is the pattern for distributing work across a pool of workers.

```
PUSH ──── task ────> PULL (worker 1)
     ──── task ────> PULL (worker 2)
     ──── task ────> PULL (worker 3)
```

No acknowledgment. No redelivery. If a worker dies after receiving a task, that task is lost. You build your own retry logic, or you accept the loss. PUSH/PULL is for throughput, not reliability.

**DEALER/ROUTER (Advanced Request-Reply)**

`DEALER` and `ROUTER` are the asynchronous, more flexible versions of `REQ` and `REP`. A `DEALER` socket does asynchronous round-robin send and fair-queued receive. A `ROUTER` socket prepends an identity frame to each message, allowing you to route replies back to specific peers.

This pair is the basis for most real-world ZeroMQ architectures. You can build broker-like intermediaries, load balancers, and complex routing topologies using DEALER/ROUTER combinations. The trade-off is complexity — you are managing identity frames and routing logic yourself.

**PAIR**

One-to-one bidirectional communication. Used almost exclusively for inter-thread communication within a single process. Not useful for network communication.

### The Wire Protocol: ZMTP

ZeroMQ defines its own wire protocol, ZMTP (ZeroMQ Message Transport Protocol). The current version is ZMTP 3.1. It handles:

- Connection handshake and version negotiation
- Security mechanism negotiation (NULL, PLAIN, CURVE)
- Message framing (messages are composed of one or more frames)
- Heartbeating (via socket options, not protocol-level)

ZMTP is simple and efficient. Messages are length-prefixed frames, with a flag byte indicating whether more frames follow (multipart messages). There is no content-type, no headers (beyond what you put in the frames), and no metadata. If you need application-level framing, you build it yourself — typically with Protocol Buffers, MessagePack, or JSON in the message payload.

### Transport Mechanisms

ZeroMQ supports multiple transports, selectable via URL scheme:

- **`tcp://`**: TCP sockets. The workhorse for network communication.
- **`ipc://`**: Unix domain sockets. Faster than TCP for same-machine communication. Not available on Windows.
- **`inproc://`**: In-process (inter-thread). The fastest option — essentially lock-free queue passing pointers between threads.
- **`pgm://` and `epgm://`**: Pragmatic General Multicast. For reliable multicast scenarios. Rarely used in practice.

The ability to use the same API for inter-thread, inter-process, and inter-machine communication is one of ZeroMQ's most elegant features. You can prototype with `inproc://`, test with `ipc://`, and deploy with `tcp://` — same code, different connection string.

---

## Strengths

**Performance.** ZeroMQ is breathtakingly fast. With no broker in the path, message latency is bounded by network latency plus a small constant for framing and buffering. Throughput in the millions of messages per second is achievable with `inproc://` transport, and hundreds of thousands per second over TCP is routine. For latency-sensitive applications, the absence of a broker hop is not a micro-optimisation — it is a fundamental architectural advantage.

**Zero-Copy.** ZeroMQ uses zero-copy techniques where possible, particularly for large messages. The `zmq_msg_t` API allows you to pass data without copying it through the library's internals. Combined with the `inproc://` transport, this enables inter-thread messaging with virtually zero overhead.

**No Broker to Manage.** You cannot have a broker outage if you do not have a broker. You do not need to provision broker instances, monitor their health, manage their storage, plan their capacity, or debug their garbage collection pauses. The operational simplicity of not having infrastructure to manage is profound, and it is often underappreciated by people who have never been woken up by a PagerDuty alert about a broker running out of disk space.

**Polyglot Bindings.** The C/C++ core library has bindings for essentially every language that matters: Python (pyzmq), Java (JeroMQ or JZMq), Go, Rust, Node.js, C#, Ruby, Erlang, Haskell — the list goes on. The API is consistent across languages, so patterns you learn in Python translate directly to Go.

**Embeddable.** ZeroMQ is a library, not a service. You can embed it in desktop applications, mobile apps, embedded systems, or anywhere else you can link a C library. This makes it suitable for use cases where running a broker is impractical or impossible.

**Pattern Building Blocks.** The socket types are composable. You can combine them to build sophisticated distributed computing patterns — brokerless pub-sub with discovery, load-balanced worker pools, multi-hop request routing, service-oriented architectures. The zguide documents dozens of these patterns with working code. The building-block approach gives you more flexibility than any broker, at the cost of more responsibility.

---

## Weaknesses

**No Persistence.** When a message is sent, it lives in memory buffers at the sender, receiver, or in transit. If a process crashes, buffered messages are lost. If a subscriber is offline, messages sent during its absence are gone. There is no journal, no log, no replay. If you need persistence, you build it yourself (by adding a database, a WAL, or — ironically — a broker).

**No Delivery Guarantees.** ZeroMQ provides *at-most-once* delivery by default, and achieving *at-least-once* requires careful application-level design (acknowledgments, retries, idempotency). *Exactly-once* is, as always, a distributed systems fairy tale, but ZeroMQ does not even try to approximate it at the library level. You get "best effort" and a pat on the back.

**"Some Assembly Required."** This is the fundamental trade-off. ZeroMQ gives you the lego bricks; you build the house. Need service discovery? Build it. Need dead-letter handling? Build it. Need message schemas? Build it. Need monitoring? Build it. Need authentication beyond basic CURVE? Build it. For small teams or simple use cases, this assembly cost can exceed the cost of just running a broker.

**Slow Subscriber Problem.** In PUB/SUB, if a subscriber cannot keep up, the publisher's send buffer fills up and messages are dropped. There is no backpressure mechanism beyond the high-water mark (which just controls when dropping begins). For some use cases this is fine — you genuinely want to drop stale market data and only process the latest quote. For others, it is a data loss bug that you discover under load at the worst possible time.

**Discovery and Topology Management.** ZeroMQ sockets need to know where to connect. There is no built-in service discovery, no registry, no DNS-based resolution beyond what TCP gives you. In static environments this is fine. In dynamic environments (containers, auto-scaling groups), you need an external discovery mechanism. This is a solved problem (DNS, Consul, etcd), but it is one more thing you have to solve.

**Debugging Complexity.** When something goes wrong in a broker-based system, you can inspect the broker — look at queue depths, examine messages, check consumer lag. With ZeroMQ, the "broker" is distributed across every process. Debugging a misbehaving PUB/SUB topology requires instrumenting your application, because there is no central point to inspect. The library is opaque by design.

---

## Ideal Use Cases

**Inter-Process Communication.** ZeroMQ's `ipc://` and `inproc://` transports make it exceptional for communication between processes or threads on the same machine. If you are building a pipeline of processes that need to pass data efficiently, ZeroMQ is the obvious choice.

**High-Frequency Trading Infrastructure.** The latency characteristics of ZeroMQ — no broker hop, zero-copy, minimal framing overhead — make it popular in financial technology. Market data distribution, order routing, and inter-component communication in trading systems frequently use ZeroMQ or its derivatives.

**Custom Protocols.** If you are building a bespoke distributed system with specific communication requirements, ZeroMQ provides the transport layer without imposing an application protocol. You design your protocol; ZeroMQ handles the plumbing.

**Embedded and Edge Computing.** When you cannot or do not want to run a broker — on embedded devices, in edge computing nodes, in desktop applications — ZeroMQ gives you distributed messaging without infrastructure.

**Internal Microservice Communication.** Within a controlled network where you have stable addressing and can tolerate the operational overhead of managing topologies, ZeroMQ can provide extremely efficient service-to-service communication.

### When to Avoid It

**When you need persistence.** If messages must survive process restarts, use a broker.

**When you need delivery guarantees.** If losing a message is unacceptable and you do not want to build your own acknowledgment/retry system, use a broker.

**When operational simplicity matters.** If your team does not want to build and maintain custom messaging infrastructure, use a broker. This is not a criticism — it is a perfectly rational trade-off.

**When you need observability out of the box.** If you want dashboards showing message rates, consumer lag, and queue depths without building them yourself, use a broker.

---

## Operational Reality

The operational reality of ZeroMQ is paradoxical: there is less infrastructure to manage but more application-level complexity to get right.

**Deployment.** There is nothing to deploy besides your application and the linked library. This is genuinely liberating. No broker cluster to provision, no configuration to manage, no state to back up. Your CI/CD pipeline deploys your application, and the messaging comes along for free.

**Monitoring.** There is nothing to monitor besides your application. ZeroMQ does not expose metrics. There is no dashboard. If you want to know message rates, queue depths, or error counts, you instrument your application code. Most teams end up building a small monitoring layer that tracks messages sent/received per socket and exports to Prometheus or similar.

**Failure Handling.** ZeroMQ handles transient network failures gracefully — sockets automatically reconnect, and messages are queued during brief disconnections. Permanent failures (process crashes, machine failures) result in message loss for anything in the send/receive buffers. Your application needs to handle this, typically through heartbeating, timeouts, and application-level acknowledgments.

**Security.** ZeroMQ supports CurveZMQ (based on CurveCP and NaCl/libsodium) for encryption and authentication. It is actually a well-designed security mechanism — mutual authentication, perfect forward secrecy, and resistance to replay attacks. The downside is that it requires distributing and managing public keys, which is its own operational challenge.

**Versioning and Compatibility.** ZMTP includes version negotiation, so different versions of ZeroMQ can interoperate to a degree. However, the binding libraries vary in quality and version support. PyZMQ is excellent. Some other language bindings lag behind. JeroMQ (the pure Java implementation) and JZMq (the JNI wrapper) have different performance characteristics and compatibility guarantees.

---

## Code Examples

### Python (pyzmq) — Publish-Subscribe

```python
# publisher.py
import zmq
import json
import time

context = zmq.Context()
socket = context.socket(zmq.PUB)
socket.bind("tcp://*:5556")

# Give subscribers time to connect (the slow joiner problem)
time.sleep(1)

events = [
    {"type": "OrderPlaced", "orderId": "ord-001", "amount": 99.99},
    {"type": "PaymentProcessed", "orderId": "ord-001", "status": "success"},
    {"type": "OrderPlaced", "orderId": "ord-002", "amount": 249.50},
    {"type": "OrderShipped", "orderId": "ord-001", "carrier": "DHL"},
]

for event in events:
    # Topic is the prefix used for subscription filtering
    topic = event["type"]
    payload = json.dumps(event)
    socket.send_multipart([
        topic.encode("utf-8"),
        payload.encode("utf-8")
    ])
    print(f"Published: {topic}")

socket.close()
context.term()
```

```python
# subscriber.py
import zmq
import json

context = zmq.Context()
socket = context.socket(zmq.SUB)
socket.connect("tcp://localhost:5556")

# Subscribe to OrderPlaced events only
socket.subscribe(b"OrderPlaced")

print("Listening for OrderPlaced events...")

while True:
    try:
        topic, payload = socket.recv_multipart()
        event = json.loads(payload.decode("utf-8"))
        print(f"Received [{topic.decode()}]: {event}")
    except KeyboardInterrupt:
        break

socket.close()
context.term()
```

### Python (pyzmq) — Pipeline (PUSH/PULL)

```python
# ventilator.py — distributes tasks to workers
import zmq
import json

context = zmq.Context()
sender = context.socket(zmq.PUSH)
sender.bind("tcp://*:5557")

# Wait for workers to connect
input("Press Enter when workers are ready...")

tasks = [
    {"task_id": i, "work": f"process_item_{i}"}
    for i in range(100)
]

for task in tasks:
    sender.send_json(task)

print(f"Sent {len(tasks)} tasks")
sender.close()
context.term()
```

```python
# worker.py — pulls tasks, processes them, pushes results
import zmq
import time
import os

context = zmq.Context()

receiver = context.socket(zmq.PULL)
receiver.connect("tcp://localhost:5557")

sender = context.socket(zmq.PUSH)
sender.connect("tcp://localhost:5558")

pid = os.getpid()
print(f"Worker {pid} ready")

while True:
    task = receiver.recv_json()
    # Simulate work
    time.sleep(0.01)
    result = {
        "task_id": task["task_id"],
        "worker": pid,
        "status": "complete"
    }
    sender.send_json(result)
    print(f"Worker {pid} completed task {task['task_id']}")
```

```python
# sink.py — collects results
import zmq

context = zmq.Context()
receiver = context.socket(zmq.PULL)
receiver.bind("tcp://*:5558")

completed = 0
while completed < 100:
    result = receiver.recv_json()
    completed += 1
    if completed % 10 == 0:
        print(f"Completed {completed}/100 tasks")

print("All tasks complete")
receiver.close()
context.term()
```

### C — Request-Reply

```c
// server.c — REP socket
#include <zmq.h>
#include <string.h>
#include <stdio.h>

int main(void) {
    void *context = zmq_ctx_new();
    void *responder = zmq_socket(context, ZMQ_REP);
    zmq_bind(responder, "tcp://*:5555");

    printf("Server listening on port 5555...\n");

    while (1) {
        char buffer[256];
        int size = zmq_recv(responder, buffer, sizeof(buffer) - 1, 0);
        if (size == -1) break;
        buffer[size] = '\0';

        printf("Received: %s\n", buffer);

        // Process and reply
        const char *reply = "{\"status\": \"ok\"}";
        zmq_send(responder, reply, strlen(reply), 0);
    }

    zmq_close(responder);
    zmq_ctx_destroy(context);
    return 0;
}

// client.c — REQ socket
#include <zmq.h>
#include <string.h>
#include <stdio.h>

int main(void) {
    void *context = zmq_ctx_new();
    void *requester = zmq_socket(context, ZMQ_REQ);
    zmq_connect(requester, "tcp://localhost:5555");

    for (int i = 0; i < 10; i++) {
        char request[128];
        snprintf(request, sizeof(request),
                 "{\"action\": \"lookup\", \"id\": %d}", i);

        printf("Sending: %s\n", request);
        zmq_send(requester, request, strlen(request), 0);

        char reply[256];
        int size = zmq_recv(requester, reply, sizeof(reply) - 1, 0);
        reply[size] = '\0';
        printf("Reply: %s\n", reply);
    }

    zmq_close(requester);
    zmq_ctx_destroy(context);
    return 0;
}
```

Compile with:
```bash
gcc -o server server.c -lzmq
gcc -o client client.c -lzmq
```

---

## nanomsg and nng: The Spiritual Successors

After Martin Sustrik departed the ZeroMQ project, he created **nanomsg** — a reimagining of ZeroMQ's ideas with a cleaner C API, a more permissive license (MIT vs ZeroMQ's LGPL), and a focus on simplicity. The socket types are called "scalability protocols" and are formally specified as RFCs.

nanomsg addressed several ZeroMQ pain points:
- Simpler API (fewer footguns)
- No context object (sockets are self-contained)
- Better error handling semantics
- Pluggable transports

However, nanomsg never achieved ZeroMQ's community size or ecosystem breadth. It was technically interesting but practically niche.

**nng** (nanomsg Next Generation), also by Sustrik and later maintained by Garrett D'Amore, is the successor to nanomsg. It features an asynchronous I/O model, better thread safety, and a more modern architecture. nng is the most actively maintained of the three, but its community remains small relative to ZeroMQ.

The practical advice: if you are starting a new project and the ZeroMQ philosophy appeals to you, evaluate nng alongside ZeroMQ. nng has a cleaner API and a more modern design. ZeroMQ has a vastly larger ecosystem, more documentation, and more people who know how to use it. In most cases, the ecosystem advantage wins.

---

## Verdict

ZeroMQ is not a message broker, and judging it as one misses the point entirely. It is a networking library that solves a different problem: how do you build distributed applications with efficient, pattern-based messaging without introducing infrastructure dependencies?

If you need a library for blazing-fast inter-process communication, if you are building custom distributed systems, if you are working in an environment where running a broker is impractical, or if you want absolute control over your messaging topology — ZeroMQ is exceptional. It has earned its reputation in HFT, scientific computing, and infrastructure tooling.

If you need persistence, delivery guarantees, consumer groups, dead-letter queues, or operational visibility without building them yourself — ZeroMQ is the wrong tool. Not because it is bad, but because those features require the very infrastructure it was designed to eliminate.

The "some assembly required" warning is real and should be taken seriously. ZeroMQ gives you the components to build anything, but building anything is exactly what you will have to do. For teams with strong systems programming skills and specific performance requirements, this is empowering. For teams that want to focus on business logic and treat messaging as a commodity, it is a burden.

Choose ZeroMQ when you know exactly what you need and are prepared to build it. Choose a broker when you want someone else to have already built it for you. There is no shame in either choice, only in choosing the wrong one for your situation and then complaining about it on the internet.

Pieter Hintjens built something genuinely original. Read the zguide, even if you never use ZeroMQ in production. It will make you a better distributed systems engineer.
