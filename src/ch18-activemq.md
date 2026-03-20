# Apache ActiveMQ and Artemis

Every technology ecosystem has its veterans — the projects that were solving real problems before the current generation of engineers had written their first `Hello, World`. In the Java messaging world, Apache ActiveMQ is that veteran. It has been faithfully shuttling JMS messages since 2004, has survived multiple hype cycles, outlasted several "next big things," and remains in production at more enterprises than most people realise. It is not glamorous. It does not trend on Hacker News. It just works, mostly, for a very specific and enduring set of use cases.

Then there is Artemis — the younger, faster, architecturally superior successor that everyone agrees is the future but that still lives in the shadow of Classic's installed base. The relationship between the two is a case study in how difficult it is to deprecate enterprise software, even when you have built something unambiguously better.

This chapter covers both, because in practice you cannot understand one without the other.

---

## Overview

### ActiveMQ Classic

ActiveMQ was created in 2004 by a group of developers who needed an open-source JMS broker. At the time, the alternatives were proprietary and expensive — IBM MQ (then WebSphere MQ), TIBCO EMS, SonicMQ. The Java Message Service specification existed, but open-source implementations were thin on the ground. ActiveMQ filled that gap.

It became an Apache top-level project in 2005 and quickly established itself as *the* open-source JMS broker. If you were building enterprise Java applications in the mid-2000s and needed messaging, ActiveMQ was likely on your shortlist. It was embedded in Apache ServiceMix, used by Apache Camel, and became a cornerstone of the enterprise integration landscape.

Classic is built on a traditional architecture: a broker process that accepts connections, routes messages to queues and topics, and manages persistence through KahaDB (its default storage engine). It works. It has worked for twenty years. But the architecture has limits that become apparent at scale, and those limits are why Artemis exists.

### Apache ActiveMQ Artemis

Artemis has a more interesting lineage than its name suggests. It started life as JBoss Messaging, was rewritten as HornetQ by the JBoss/Red Hat team, and was then donated to the Apache Foundation in 2015 to become the next generation of ActiveMQ. The person most associated with the project is Clebert Suconic, who led HornetQ and continued to shepherd Artemis.

The donation was not purely altruistic. Red Hat had a message broker (HornetQ) that was excellent but had a small community relative to ActiveMQ's name recognition. Apache had a message broker (ActiveMQ Classic) with massive name recognition but an aging architecture. The merger made sense on paper, and — unusually for these things — it has largely worked in practice.

Artemis is not a patched version of Classic. It is a ground-up rewrite with a fundamentally different architecture. The two share a name, a community, and a set of goals, but very little code.

---

## Architecture

### ActiveMQ Classic Architecture

Classic follows the traditional enterprise broker pattern. A JVM process runs the broker. Clients connect via one of several supported protocols. Messages are received, persisted (if durable), routed to the appropriate destination, and delivered to consumers. The storage engine (KahaDB by default, though JDBC-backed storage is available) handles persistence.

The architecture is straightforward but single-threaded in some critical paths. KahaDB uses a transaction journal plus B-tree index approach that works well at moderate volumes but can become a bottleneck under heavy load. The broker maintains an in-memory dispatch queue and pages to disk when memory limits are reached, which can introduce unpredictable latency spikes.

Classic's networking model supports "networks of brokers" — multiple broker instances connected in a mesh or tree topology to distribute load. This works, but it is operationally complex, the forwarding logic has historically been a source of subtle bugs, and the semantics of message ordering across a network of brokers are... best described as "approximate."

### Artemis Architecture

Artemis takes a meaningfully different approach, and the differences matter.

**Journal-Based Storage.** Artemis uses an append-only journal for persistence — a design that should sound familiar to anyone who has spent time with Kafka or any modern log-structured storage system. The journal writes are sequential, which means they can saturate disk I/O bandwidth far more efficiently than the random I/O patterns of Classic's KahaDB. The journal supports both NIO (Java NIO file channels) and AIO (Linux asynchronous I/O via libaio) backends. On Linux with AIO, write performance is exceptional.

```
# Artemis journal configuration (broker.xml)
<journal-type>ASYNCIO</journal-type>
<journal-directory>data/journal</journal-directory>
<journal-min-files>2</journal-min-files>
<journal-pool-files>10</journal-pool-files>
<journal-file-size>10M</journal-file-size>
<journal-buffer-timeout>4000</journal-buffer-timeout>
```

**Non-Blocking I/O.** Artemis uses Netty for all network I/O, which means it can handle far more concurrent connections than Classic on the same hardware. The threading model is designed around a small number of I/O threads feeding work to a configurable thread pool, avoiding the thread-per-connection pattern that limited Classic.

**Paging.** When destinations exceed their configured memory limits, Artemis pages messages to disk transparently. Unlike Classic's approach (which could lead to unpredictable behaviour), Artemis paging is a first-class feature with well-defined semantics. Messages are paged in order, consumers drain the in-memory messages first, then paged messages are brought back in order. It is not magic — paging adds latency — but it is predictable.

**Large Message Support.** Messages that exceed a configurable threshold (default 100KB) are stored outside the journal in separate files. This prevents large payloads from bloating the journal and affecting throughput for smaller messages. A practical feature that reflects real-world usage patterns where the occasional 50MB message coexists with millions of 1KB messages.

**Address Model.** Artemis uses a unified address model that is more flexible than the traditional JMS queue/topic distinction. An *address* is a named endpoint. Attached to each address are one or more *queues*. The routing type determines behaviour:

- **Anycast**: messages are distributed across queues in a round-robin fashion (queue semantics)
- **Multicast**: messages are copied to all queues (topic semantics)

This model cleanly maps to JMS queues and topics, AMQP links, MQTT subscriptions, and STOMP destinations. It is one of the reasons Artemis can support so many protocols simultaneously without awkward impedance mismatches.

---

## Protocol Support

This is one of ActiveMQ's genuine differentiators — both Classic and Artemis are protocol polyglots.

### Classic Protocols
- **OpenWire**: The native ActiveMQ wire protocol, used by the ActiveMQ JMS client. Efficient for Java-to-Java communication.
- **STOMP**: Simple text-based protocol. Good for non-Java clients.
- **AMQP 1.0**: Added later, but functional.
- **MQTT**: For IoT use cases.
- **WebSocket**: For browser-based clients.

### Artemis Protocols
- **Core**: Artemis's native high-performance protocol.
- **OpenWire**: For backward compatibility with existing ActiveMQ Classic clients.
- **AMQP 1.0**: First-class support, not an afterthought.
- **STOMP**: Text-based simplicity.
- **MQTT**: Versions 3.1, 3.1.1, and 5.0.
- **HornetQ**: For migration from HornetQ installations.

Each protocol runs on its own acceptor (port), or multiple protocols can share a single port with auto-detection. The fact that you can have a Java JMS producer sending via OpenWire, a Python consumer receiving via AMQP, and an IoT device publishing via MQTT — all through the same broker, all interoperating on the same address — is genuinely useful in heterogeneous environments. It is also the kind of thing that makes debugging exciting in ways you did not ask for.

---

## Clustering

### Classic: Network of Brokers

Classic's clustering model involves connecting multiple brokers via "network connectors." Brokers forward messages to each other based on consumer demand. If broker A has a message for a queue, and broker B has a consumer for that queue, the message is forwarded.

This model has the advantage of conceptual simplicity and the disadvantage of operational complexity. Message ordering across the network is not guaranteed. Duplicate detection requires care. Network splits can lead to message duplication or loss, depending on your configuration. The "advisory message" system (used for internal broker-to-broker communication) can itself become a performance bottleneck. Enterprises have made this work, often with dedicated messaging teams who understand the failure modes intimately.

### Artemis: Live-Backup Pairs and Clustering

Artemis takes a different approach to high availability and clustering:

**Live-Backup Pairs.** A primary (live) server has one or more backup servers. The backup replicates the journal from the primary. If the primary fails, the backup activates and takes over. Failover is automatic for clients using the Artemis or OpenWire client libraries. This is straightforward HA — no split-brain if you configure the quorum correctly, deterministic failover, and the backup has a warm copy of the data.

```xml
<!-- Primary broker configuration -->
<ha-policy>
  <replication>
    <primary>
      <group-name>my-pair</group-name>
      <vote-on-replication-failure>true</vote-on-replication-failure>
      <quorum-size>1</quorum-size>
    </primary>
  </replication>
</ha-policy>

<!-- Backup broker configuration -->
<ha-policy>
  <replication>
    <backup>
      <group-name>my-pair</group-name>
      <allow-failback>true</allow-failback>
    </backup>
  </replication>
</ha-policy>
```

**Clustering (Symmetric Cluster).** Multiple live-backup pairs can be connected in a cluster. Unlike Classic's network of brokers, Artemis clusters use a more formal message redistribution mechanism. Messages are redistributed between nodes when consumers exist on other nodes, and the redistribution delay is configurable to allow local consumers a chance to process messages first.

The clustering model works well but is not a replacement for something like Kafka's partition-based parallelism. Artemis clustering is designed for HA and moderate scale-out, not for the kind of massive horizontal scaling that Kafka enables. Know what it is designed for, and you will not be disappointed.

---

## Strengths

**JMS Compliance.** If you need a JMS broker, ActiveMQ (either variant) is one of the most complete implementations available. Artemis passes the JMS TCK, supports JMS 2.0, and handles the full range of JMS features — selectors, message groups, scheduled delivery, last-value queues, transactions (including XA). This matters less than it used to, but in environments where JMS is a requirement (and there are more of these than Twitter would have you believe), it matters a lot.

**Protocol Polyglot.** Supporting AMQP, MQTT, STOMP, OpenWire, and a native protocol on the same broker is genuinely useful. You do not need separate infrastructure for your Java services, your Python scripts, and your IoT devices. One broker, multiple protocols, shared destinations.

**Enterprise Integration.** ActiveMQ is a natural fit with Apache Camel, Spring Boot, and the broader Java enterprise ecosystem. The integration is deep, well-documented, and battle-tested. Spring's `JmsTemplate`, Camel's JMS component, and Java EE's MDB (Message-Driven Bean) pattern all work seamlessly.

**Maturity.** Twenty years of production deployments have surfaced and fixed a lot of bugs. The failure modes are well-understood. The documentation, while not always exciting, is comprehensive. The mailing list archives are a treasure trove of solutions to problems you have not encountered yet.

**Artemis Performance.** Artemis is genuinely fast for a traditional broker. The journal-based storage, non-blocking I/O, and efficient threading model combine to deliver throughput and latency numbers that are competitive with anything in the traditional broker category. It will not match Kafka for raw throughput on append-only workloads, but for transactional, routed messaging it is excellent.

---

## Weaknesses

**Java Ecosystem Lock-In.** ActiveMQ is a Java application. Its management tools are Java. Its plugin system is Java. Its best client library is Java. Yes, other protocols provide non-Java access, but the centre of gravity is firmly in the JVM world. If your organisation is primarily Python, Go, or Rust, ActiveMQ will feel like a foreign object.

**Throughput Ceiling.** For all of Artemis's improvements, it is still a traditional broker — every message passes through the broker, is persisted, and is dispatched. For workloads in the millions-of-messages-per-second range, you need Kafka, Redpanda, or a log-based system. Artemis targets the tens-of-thousands to low-hundreds-of-thousands messages per second range (per broker), which is plenty for most enterprise workloads but not enough for high-volume event streaming.

**Community Momentum.** The messaging conversation has shifted to Kafka, Pulsar, NATS, and cloud-native alternatives. ActiveMQ development continues, but the community is smaller and less active than it was a decade ago. Finding experienced ActiveMQ operators under the age of forty is increasingly challenging. This is not a technical problem, but it is a practical one.

**Classic's Technical Debt.** Classic's architecture shows its age. KahaDB has known performance limitations. The thread model does not scale well with connection count. The network of brokers feature, while functional, is complex and fragile. If you are starting a new project, there is no technical reason to choose Classic over Artemis.

**Configuration Complexity.** Artemis is configured via XML files (`broker.xml`, `bootstrap.xml`, etc.) that are comprehensive but verbose. The number of tunables is vast, and the defaults are not always optimal for your workload. You will spend time reading documentation about journal buffer sizes, thread pool configurations, and address settings. This is the price of flexibility.

---

## Ideal Use Cases

**Existing Java/JMS Ecosystems.** If you have Java services that speak JMS, ActiveMQ (particularly Artemis) is a natural choice. The migration path from other JMS brokers (IBM MQ, TIBCO EMS) is well-documented.

**Protocol Bridge.** When you need a single broker that speaks AMQP, MQTT, STOMP, and JMS, Artemis is one of the few options that handles all of them competently.

**Enterprise Integration Patterns.** If your architecture involves Apache Camel routes, Enterprise Service Bus patterns, or traditional request-reply messaging, ActiveMQ was designed for this world.

**Transactional Messaging.** XA transactions, JMS transactions, and the full machinery of transactional messaging are first-class features. If your workflow requires "dequeue message, update database, commit both atomically," ActiveMQ has you covered.

**Gradual Modernisation.** Organisations moving from monolithic to distributed architectures can start with ActiveMQ (a familiar paradigm for enterprise Java developers) and evolve toward event streaming as their needs grow.

---

## Operational Reality

Running ActiveMQ in production is not inherently difficult, but it does require the kind of care and feeding that any stateful system demands.

**JMX Monitoring.** Both Classic and Artemis expose management and monitoring via JMX. This is the primary monitoring interface, and if you are not running a JMX-aware monitoring stack, you will need to set one up. Prometheus exporters exist (the `jmx_exporter` from Prometheus works well) but require configuration to expose the metrics you actually care about.

Key metrics to watch:
- Queue depth (message count per destination)
- Enqueue/dequeue rates
- Memory usage (broker heap, journal pages)
- Connection count
- Consumer count per destination
- DLQ (dead-letter queue) depth — this is your canary

**Memory Tuning.** Artemis is a JVM application, which means you are in the business of tuning garbage collection. For latency-sensitive workloads, G1GC or ZGC with appropriately sized heaps is the starting point. The journal buffer size, page size, and global max size all interact with JVM heap settings in ways that are not immediately intuitive. The Artemis documentation covers this well, but expect to spend an afternoon with GC logs and a profiler during initial setup.

```bash
# Typical Artemis JVM settings for a production broker
JAVA_ARGS="-Xms4g -Xmx4g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=50 \
  -XX:+ParallelRefProcEnabled \
  -XX:+UseStringDeduplication \
  -Dhawtio.realm=activemq \
  -Dhawtio.offline=true"
```

**Journal Management.** The Artemis journal will grow as messages accumulate. Paging kicks in when in-memory limits are reached. If consumers fall behind and paged data grows unbounded, you will eventually run out of disk. Monitoring disk usage and setting address-level limits (with appropriate `address-full-policy` — `PAGE`, `DROP`, `BLOCK`, or `FAIL`) is essential.

**Upgrades.** ActiveMQ upgrades are generally straightforward — the project takes backward compatibility seriously. Artemis supports rolling upgrades within minor versions. Major version upgrades require more care but are well-documented.

**Hawtio Console.** Artemis ships with the Hawtio web console for management and monitoring. It is functional if somewhat dated in appearance. You can view queue depths, browse messages, send test messages, and manage addresses. It is adequate for development and debugging but should not be your primary production monitoring tool.

---

## Code Examples

### Java JMS 2.0 (Artemis)

```java
import javax.jms.*;
import org.apache.activemq.artemis.jms.client.ActiveMQConnectionFactory;

public class ArtemisJmsExample {

    public static void main(String[] args) throws Exception {
        // Producer
        try (ActiveMQConnectionFactory factory =
                new ActiveMQConnectionFactory("tcp://localhost:61616");
             JMSContext context = factory.createContext()) {

            JMSProducer producer = context.createProducer();
            Queue queue = context.createQueue("orders");

            // Send with properties for routing/filtering
            TextMessage message = context.createTextMessage(
                "{\"orderId\": \"ord-7829\", \"amount\": 149.99}");
            message.setStringProperty("eventType", "OrderPlaced");
            message.setStringProperty("region", "eu-west");

            producer.send(queue, message);
            System.out.println("Sent: " + message.getText());
        }

        // Consumer
        try (ActiveMQConnectionFactory factory =
                new ActiveMQConnectionFactory("tcp://localhost:61616");
             JMSContext context = factory.createContext()) {

            Queue queue = context.createQueue("orders");

            // Selector: only receive EU orders
            JMSConsumer consumer = context.createConsumer(queue,
                "region = 'eu-west'");

            Message received = consumer.receive(5000);
            if (received instanceof TextMessage) {
                System.out.println("Received: " +
                    ((TextMessage) received).getText());
            }
        }
    }
}
```

### Spring Boot with JMS

```java
// Configuration
@Configuration
public class ArtemisConfig {

    @Bean
    public ConnectionFactory connectionFactory() {
        return new ActiveMQConnectionFactory("tcp://localhost:61616");
    }

    @Bean
    public JmsTemplate jmsTemplate(ConnectionFactory connectionFactory) {
        JmsTemplate template = new JmsTemplate(connectionFactory);
        template.setDefaultDestinationName("events");
        return template;
    }
}

// Producer Service
@Service
public class OrderEventPublisher {

    private final JmsTemplate jmsTemplate;

    public OrderEventPublisher(JmsTemplate jmsTemplate) {
        this.jmsTemplate = jmsTemplate;
    }

    public void publishOrderPlaced(Order order) {
        jmsTemplate.convertAndSend("orders", order, message -> {
            message.setStringProperty("eventType", "OrderPlaced");
            message.setStringProperty("correlationId",
                UUID.randomUUID().toString());
            return message;
        });
    }
}

// Consumer - Message-Driven POJO
@Component
public class OrderEventConsumer {

    private static final Logger log =
        LoggerFactory.getLogger(OrderEventConsumer.class);

    @JmsListener(destination = "orders",
                 selector = "eventType = 'OrderPlaced'")
    public void handleOrderPlaced(Order order,
                                   @Header("correlationId") String corrId) {
        log.info("Processing order {} (correlationId: {})",
            order.getOrderId(), corrId);
        // Process the order event
    }
}
```

### AMQP 1.0 (Python, using python-qpid-proton)

```python
from proton import Message
from proton.handlers import MessagingHandler
from proton.reactor import Container

class OrderProducer(MessagingHandler):
    """Sends order events to Artemis via AMQP 1.0."""

    def __init__(self, url, queue, messages):
        super().__init__()
        self.url = url
        self.queue = queue
        self.messages = messages
        self.sent = 0

    def on_start(self, event):
        conn = event.container.connect(self.url)
        self.sender = event.container.create_sender(conn, self.queue)

    def on_sendable(self, event):
        while event.sender.credit and self.sent < len(self.messages):
            msg = Message(
                body=self.messages[self.sent],
                properties={"event_type": "OrderPlaced"},
                content_type="application/json"
            )
            event.sender.send(msg)
            self.sent += 1

        if self.sent == len(self.messages):
            event.sender.close()
            event.connection.close()


class OrderConsumer(MessagingHandler):
    """Receives order events from Artemis via AMQP 1.0."""

    def __init__(self, url, queue):
        super().__init__()
        self.url = url
        self.queue = queue

    def on_start(self, event):
        conn = event.container.connect(self.url)
        event.container.create_receiver(conn, self.queue)

    def on_message(self, event):
        print(f"Received: {event.message.body}")
        print(f"Event type: {event.message.properties.get('event_type')}")


if __name__ == "__main__":
    # Artemis AMQP port is 5672 by default
    url = "amqp://localhost:5672"

    orders = [
        '{"orderId": "ord-001", "amount": 99.99}',
        '{"orderId": "ord-002", "amount": 249.50}',
    ]

    Container(OrderProducer(url, "orders", orders)).run()
    Container(OrderConsumer(url, "orders")).run()
```

---

## Classic vs Artemis: The Migration Question

If you are running Classic in production, the question is not *whether* to migrate to Artemis but *when*. Classic is in maintenance mode — it receives security fixes and critical bug fixes, but active development has moved to Artemis. The ActiveMQ project has been clear about this.

The migration is not trivial but it is well-supported:

1. **OpenWire Compatibility.** Artemis speaks OpenWire, so existing ActiveMQ Classic clients can connect to Artemis without code changes. This is the most important migration enabler.

2. **Configuration Translation.** The configuration models are different (Classic uses `activemq.xml`, Artemis uses `broker.xml`), but there is a migration tool and documentation that maps concepts between the two.

3. **Feature Parity.** Artemis supports nearly all Classic features, with some differences in semantics. Virtual topics from Classic map to Artemis's address model. Network of brokers maps to Artemis clustering (with different semantics). Message groups, selectors, and scheduled delivery all have Artemis equivalents.

4. **Behavioural Differences.** Some things work differently enough to require testing. Message priority handling, redelivery semantics, and memory management all differ in detail. Plan for a testing phase.

The practical advice: if Classic is working and you have no pressing issues, plan the migration but do not rush it. If you are starting a new project, use Artemis. There is no remaining reason to start with Classic.

---

## Verdict

ActiveMQ Artemis is a thoroughly competent message broker that does not get the attention it deserves. It is fast, reliable, supports more protocols than almost anything else in the space, and has the kind of deep JMS support that enterprises actually need. It is not trying to be Kafka — it is not a distributed log, it is not designed for massive horizontal scaling, and it does not pretend to be a streaming platform.

What it *is* is a very good traditional message broker. If your use case involves routing messages between services, supporting mixed protocols, transactional messaging, or enterprise integration patterns, Artemis is a strong choice. If you are already in the Java ecosystem, it is an excellent one.

The honest risk is organisational, not technical. The community is smaller than Kafka's or RabbitMQ's. Hiring expertise is harder. The project moves at an enterprise pace, which means stability but slower innovation. If you choose Artemis, you are betting on a project with solid technical foundations and a dedicated (if small) maintainer community.

For new Java enterprise projects that need a message broker: Artemis is a top-tier choice. For polyglot microservices architectures: consider NATS or RabbitMQ first. For event streaming at scale: Kafka or Redpanda. For legacy Classic installations: start planning the Artemis migration, and take comfort in the fact that it is one of the smoother migration paths in the messaging world.

ActiveMQ may not be exciting, but in infrastructure, "exciting" is rarely a compliment.
