# Redpanda

There is a particular category of startup pitch that goes like this: "It's like [widely adopted thing], but we rewrote it in [faster language] and removed [thing everyone complains about]." Most of these pitches produce vapourware. Occasionally, one produces something genuinely good. Redpanda is in the latter category.

Redpanda's proposition is simple and audacious: a Kafka-compatible streaming platform, rewritten from scratch in C++, with no JVM, no ZooKeeper, and dramatically simpler operations. It is the kind of project that makes Kafka administrators feel a complex mix of excitement and professional anxiety.

---

## Overview

Redpanda was founded as Vectorized, Inc. by Alexander Gallego in 2019. Gallego, previously at Akamai, had experience with high-performance C++ systems and a conviction that Kafka's performance limitations were largely artefacts of the JVM and ZooKeeper, not fundamental to the log-based streaming model. The company rebranded to Redpanda Data in 2021.

The core thesis: Kafka's *design* is sound — append-only logs, consumer groups, partitioned topics — but its *implementation* leaves performance on the table. The JVM introduces GC pauses. ZooKeeper adds operational complexity and a coordination bottleneck. The multi-threaded architecture contends on shared data structures. A C++ implementation using a thread-per-core model (via the Seastar framework) could deliver the same semantics with lower latency, higher throughput, and fewer moving parts.

This is not just a theory. Redpanda has delivered on enough of this promise to be taken seriously, while also encountering the predictable challenges of building a compatible replacement for a system with a decade-long head start.

The company has raised significant venture capital funding, launched a managed cloud offering, and attracted adoption from organisations that want Kafka's model without Kafka's operational overhead. Whether Redpanda will become a dominant platform or remain an excellent niche alternative is still an open question, but it has already influenced the broader ecosystem — Kafka's own move away from ZooKeeper (KRaft) was, shall we say, coincidentally well-timed.

---

## Architecture

### Thread-Per-Core (Seastar)

Redpanda is built on the Seastar framework, the same C++ framework that powers ScyllaDB. Seastar's model is thread-per-core: each CPU core runs a single thread (called a "shard"), and each shard owns its own memory, its own network connections, and its own data. Shards communicate via explicit message passing, not shared memory.

This eliminates lock contention, reduces context switching, and makes performance predictable. A 16-core machine runs 16 independent shards, each processing its own subset of partitions. There is no global lock, no shared heap, and no garbage collector deciding that *now* would be a great time to pause for 200 milliseconds.

The practical result is lower tail latency. Kafka's P99 latency can spike during GC pauses or under load as threads contend on shared data structures. Redpanda's P99 is more consistent because the architecture eliminates the primary sources of jitter.

### Raft Consensus (No ZooKeeper)

Where Kafka historically depended on ZooKeeper for metadata management, leader election, and coordination, Redpanda uses Raft consensus internally. Every partition has a Raft group. Metadata is managed by an internal Raft group. There is no external dependency.

This is significant operationally. ZooKeeper was Kafka's most notorious operational pain point — a separate distributed system with its own failure modes, its own monitoring requirements, and its own capacity planning needs. "We upgraded ZooKeeper and the Kafka cluster fell over" is a story that has been told at more post-mortems than anyone would like to admit.

Redpanda runs as a single binary. Start the binary, join the cluster, done. This is not just marketing simplicity — it materially reduces the surface area for operational failures.

It is worth noting that Kafka has been addressing this with KRaft (Kafka Raft), which replaces ZooKeeper with an internal Raft-based metadata quorum. KRaft reached production readiness in Kafka 3.3 and ZooKeeper mode was formally deprecated. The competitive pressure from Redpanda was almost certainly a factor in accelerating this work, even if no one will say so on the record.

### Storage

Redpanda uses a custom storage engine optimised for append-only workloads. Data is written to a write-ahead log, and the storage layer is designed to exploit sequential I/O patterns. The system is self-tuning — it detects the underlying storage characteristics (SSD vs HDD, local vs network-attached) and adjusts its I/O scheduling accordingly.

**Tiered Storage** allows older segments to be offloaded to object storage (S3, GCS, Azure Blob Storage). This is the same concept as Kafka's tiered storage (KIP-405), and it addresses the cost problem of keeping large retention periods on fast local storage. Hot data stays on local SSDs; cold data moves to cheap object storage and is fetched on demand.

Tiered storage is a significant feature for cost management. If you are retaining 30 days of data but most reads are from the last 2 hours, paying for SSD storage for the full 30 days is wasteful. Tiered storage lets you size local disks for the hot set and let S3 handle the rest.

### Self-Tuning

Redpanda performs automatic tuning during startup: it benchmarks the disks, detects the CPU topology, configures I/O scheduling parameters, and sets buffer sizes. The `rpk` (Redpanda Keeper) CLI tool includes a tuning mode that configures the operating system as well — setting CPU governor, disabling IRQ balancing, configuring huge pages.

```bash
# Auto-tune the system
sudo rpk redpanda tune all

# Check tuning status
rpk redpanda tune list
```

This is a direct response to one of Kafka's operational realities: achieving optimal performance requires manual tuning of dozens of parameters (JVM heap, GC settings, `num.io.threads`, `log.flush.interval.messages`, OS-level settings). Redpanda's position is that the system should figure this out itself. The auto-tuning is not perfect — you may still need to adjust settings for unusual workloads — but it dramatically reduces the time from installation to acceptable performance.

---

## Kafka API Compatibility

This is Redpanda's most strategically important feature: it implements the Kafka wire protocol. Kafka clients — the Java client, librdkafka, franz-go, Sarama, confluent-kafka-python — connect to Redpanda as though it were Kafka. No code changes required.

### What Works

- **Core produce/consume operations.** The bread and butter. Producing and consuming messages with standard Kafka clients works as expected.
- **Consumer groups.** Group coordination, offset management, rebalancing — all implemented.
- **Transactions.** Exactly-once semantics (EOS) with idempotent producers and transactional consumers.
- **ACLs.** Kafka's authorisation model is supported.
- **Schema Registry.** Redpanda includes a built-in Schema Registry that is compatible with the Confluent Schema Registry API. Avro, Protobuf, and JSON Schema are supported.
- **Admin API.** Topic creation, configuration changes, partition reassignment.
- **Kafka Connect.** Because Connect uses the standard Kafka protocol, existing connectors work with Redpanda. Though Redpanda does not bundle Connect itself — you run the Kafka Connect framework pointed at Redpanda.

### What Does Not (or Did Not, or Requires Caveats)

- **Kafka Streams.** Kafka Streams is a client library that uses internal topics and specific protocol features. It works with Redpanda, but compatibility is "best effort" rather than guaranteed. Simple Streams applications work fine; complex topologies with state stores may hit edge cases.
- **Exactly matching Kafka's behaviour in all edge cases.** The protocol specification leaves room for implementation-defined behaviour. Redpanda aims for compatibility, but subtle differences exist — particularly in error handling, retry semantics, and edge cases around partition rebalancing. Most applications will never encounter these differences. If you are relying on undocumented Kafka behaviour, test carefully.
- **MirrorMaker 2.** Works, but Redpanda also provides its own migration tooling for moving data from Kafka to Redpanda.
- **Some newer Kafka protocol versions.** Redpanda tracks Kafka protocol versions but sometimes lags the latest release by a few months. Check the compatibility matrix for your specific Kafka client version.

The practical reality is that for most applications, "just point your Kafka clients at Redpanda" works. This is an extraordinary engineering achievement and the primary reason Redpanda has gained traction. The switching cost is near zero for the common case.

---

## Console

Redpanda Console (formerly Kowl, which Redpanda acquired) is a web-based UI for managing and monitoring Redpanda (and Kafka) clusters. It provides:

- Topic browsing and message inspection
- Consumer group monitoring (lag, offsets)
- Schema Registry management
- ACL management
- Cluster health overview

The Console is noticeably more polished than most open-source Kafka UIs. It is included with Redpanda and also works with vanilla Kafka clusters, which is a clever way to let people evaluate Redpanda's ecosystem before migrating.

---

## Strengths

**Lower Latency.** Redpanda's thread-per-core architecture delivers consistently lower tail latency than Kafka. For workloads where P99 latency matters — real-time applications, interactive systems, latency-sensitive pipelines — this is a genuine advantage. The improvement is most pronounced under load, where Kafka's GC pauses and thread contention become noticeable and Redpanda's architecture avoids them entirely.

**Simpler Operations.** Single binary. No ZooKeeper. No JVM tuning. Self-tuning I/O. This is not just a convenience — it reduces the mean time to recovery, lowers the skill barrier for operations teams, and shrinks the surface area for configuration-related outages. If you have ever spent a day debugging a Kafka cluster that fell over because ZooKeeper ran out of ephemeral nodes, you will appreciate the simplicity viscerally.

**Kafka Compatibility.** Using existing Kafka clients with no code changes is a massive advantage for adoption. It means you can evaluate Redpanda without rewriting anything, migrate incrementally, and fall back to Kafka if needed. The risk of trying Redpanda is low, which makes the decision to evaluate it easy.

**Resource Efficiency.** Redpanda typically requires fewer nodes than Kafka for the same workload, and does not need the additional ZooKeeper nodes. The C++ implementation uses memory more efficiently than the JVM — no GC overhead, no object header overhead, deterministic allocation. For cloud deployments, fewer nodes means lower costs.

**Built-in Schema Registry.** Having the Schema Registry integrated (rather than requiring a separate Confluent Schema Registry deployment) simplifies the architecture. One fewer service to deploy, monitor, and manage.

**`rpk` CLI.** The `rpk` command-line tool is well-designed. Topic management, cluster configuration, performance testing, and system tuning are all accessible from a single, coherent CLI. It is the kind of developer experience that suggests the team has actually used their own product.

---

## Weaknesses

**Younger Ecosystem.** Kafka has been in production since 2011. It has been battle-tested at LinkedIn, Netflix, Uber, and thousands of other organisations. The Kafka ecosystem includes Kafka Streams, ksqlDB, Kafka Connect with hundreds of connectors, and a vast body of operational knowledge. Redpanda is significantly younger. It has been in production at real companies, but the breadth and depth of real-world experience is smaller. Edge cases that Kafka has encountered and fixed over a decade may still be lurking in Redpanda.

**Smaller Community.** The community is growing but still a fraction of Kafka's. Fewer blog posts, fewer Stack Overflow answers, fewer consultants, fewer books. When you hit a problem, you are more likely to be the first person to encounter it. The Redpanda team is responsive (their Slack community is active), but community-sourced knowledge is thinner.

**Enterprise Features Behind License.** Redpanda Community Edition is open source (BSL, which converts to Apache 2.0 after four years). Enterprise features — continuous data balancing, tiered storage (in some configurations), audit logging, role-based access control beyond basic ACLs — require an Enterprise license. This is a reasonable business model, but it means the "free" Redpanda does not include everything you might need for a production deployment. Evaluate which features you need before committing.

**Kafka Compatibility Is Not Identity.** Despite the impressive compatibility, Redpanda is not Kafka. Behaviour differs in edge cases. Tools that rely on Kafka internals (rather than the public API) may not work. Monitoring tools that use Kafka's JMX metrics will not work — Redpanda exports Prometheus metrics natively (which is arguably better, but different). If you have extensive Kafka-specific operational tooling, migration requires adapting that tooling.

**Limited Ecosystem Integration.** Kafka Streams and ksqlDB work with Redpanda to varying degrees, but they are not the focus of Redpanda's development. If your architecture relies heavily on Kafka Streams state stores, interactive queries, or ksqlDB materialised views, test thoroughly before committing. Redpanda's own answer to stream processing is to integrate with Flink, Benthos, or other external processors.

---

## Ideal Use Cases

**Latency-Sensitive Streaming.** When P99 latency matters and Kafka's GC pauses are a problem you have actually measured (not just feared), Redpanda is a compelling alternative.

**Operational Simplicity.** For teams that want Kafka's semantics without the operational complexity — particularly smaller teams without dedicated Kafka operators — Redpanda's single-binary, self-tuning design is attractive.

**Kafka Migration.** If you are running Kafka and are frustrated by operational overhead, Redpanda offers a relatively low-risk migration path. The Kafka protocol compatibility means you can test with your actual applications before committing.

**Cost Optimisation.** If you are running Kafka in the cloud and the node count (Kafka brokers plus ZooKeeper nodes) is driving costs, Redpanda's resource efficiency can provide meaningful savings.

**New Streaming Projects.** If you are starting a new project that needs Kafka-style semantics, Redpanda is worth evaluating alongside Kafka. The simpler operations and lower resource requirements can accelerate time to production.

---

## Operational Reality

### Deployment

Redpanda supports deployment on bare metal, virtual machines, Kubernetes (via a Helm chart and a Kubernetes operator), and Docker. The Kubernetes operator handles cluster provisioning, scaling, and upgrades.

```bash
# Install Redpanda on Linux
curl -1sLf \
  'https://dl.redpanda.com/nzc4ZYQK3WRGd9sy/redpanda/cfg/setup/bash.deb.sh' \
  | sudo -E bash
sudo apt install redpanda

# Start and configure
sudo rpk redpanda tune all
sudo systemctl start redpanda

# Or with Docker
docker run -d --name=redpanda \
  -p 9092:9092 -p 8081:8081 -p 8082:8082 -p 9644:9644 \
  docker.redpanda.com/redpandadata/redpanda:latest \
  redpanda start --smp 1 --memory 1G --overprovisioned
```

### Monitoring

Redpanda exports Prometheus metrics natively on port 9644. No JMX exporter, no custom configuration. The metrics cover:

- Throughput (bytes/messages in/out per topic/partition)
- Latency (produce/fetch latency histograms)
- Storage (disk usage, log segment counts)
- Consumer groups (committed offsets, lag)
- Raft (leader elections, replication lag)
- Internal metrics (scheduler utilisation, memory allocation)

Grafana dashboards are available from Redpanda and the community. The metrics naming follows Prometheus conventions, which means they integrate cleanly with existing Prometheus/Grafana stacks.

### The "Just Replace Kafka" Migration Path

The migration story is approximately:

1. Deploy a Redpanda cluster alongside your existing Kafka cluster.
2. Use MirrorMaker 2 or Redpanda's own migration tool to replicate topics from Kafka to Redpanda.
3. Point consumers at Redpanda, verify they work.
4. Point producers at Redpanda.
5. Decommission Kafka.

Steps 3 and 4 are where you discover whether "Kafka compatible" means compatible enough for *your* applications. In most cases, it does. But "most cases" provides cold comfort if you are the exception.

The recommended approach is to run both systems in parallel for a meaningful period, comparing outputs, monitoring for discrepancies, and building confidence. Do not do a big-bang migration on a Friday afternoon.

### Redpanda Cloud and BYOC

**Redpanda Cloud** is the fully managed offering — Redpanda runs the infrastructure, you use it as a service. Standard managed service model.

**BYOC (Bring Your Own Cloud)** is more interesting: the data plane runs in your cloud account (your VPC, your machines), while Redpanda manages the control plane. This addresses the concern that some organisations have about sending data through a third party's infrastructure while still offloading operational management. It is a pragmatic middle ground for security-conscious organisations.

---

## Benchmarks and Performance Claims

Redpanda publishes benchmarks showing superior throughput and latency compared to Kafka. These benchmarks are produced by Redpanda, which is the first thing you should note. Vendor benchmarks are a literary genre, not a scientific discipline.

That said, independent benchmarks generally confirm two things:

1. **Latency is consistently lower**, particularly P99 and P999 tail latency. This is the thread-per-core architecture paying off, and the advantage is most visible under load.

2. **Throughput is competitive to superior**, depending on workload. For produce-heavy workloads with small messages, Redpanda often outperforms Kafka. For very large messages or workloads that are primarily sequential reads from well-cached data, the difference narrows.

The benchmarks you should trust are the ones you run yourself, on your hardware, with your workload patterns, at your scale. Redpanda provides `rpk` tooling for load testing:

```bash
# Produce benchmark
rpk topic produce benchmark-topic --compression none \
  --num 1000000 --batch-size 100

# Consume benchmark
rpk topic consume benchmark-topic --num 1000000
```

Do not make a production decision based on someone else's benchmark, including Redpanda's.

---

## Code Examples

The entire point of Kafka API compatibility is that existing Kafka clients work unchanged. These examples use standard Kafka client libraries, connecting to Redpanda instead of Kafka.

### Python (confluent-kafka-python)

```python
from confluent_kafka import Producer, Consumer, KafkaError
import json

# ---------- Producer ----------

producer_config = {
    'bootstrap.servers': 'localhost:9092',  # Redpanda address
    'client.id': 'order-service',
    'acks': 'all',
    'enable.idempotence': True,
}

producer = Producer(producer_config)

def delivery_callback(err, msg):
    if err:
        print(f"Delivery failed: {err}")
    else:
        print(f"Delivered to {msg.topic()}[{msg.partition()}] "
              f"@ offset {msg.offset()}")

event = {
    "type": "OrderPlaced",
    "orderId": "ord-7829",
    "amount": 149.99,
    "currency": "USD"
}

producer.produce(
    topic="orders",
    key="ord-7829",
    value=json.dumps(event).encode("utf-8"),
    callback=delivery_callback
)
producer.flush()

# ---------- Consumer ----------

consumer_config = {
    'bootstrap.servers': 'localhost:9092',  # Same Redpanda address
    'group.id': 'payment-service',
    'auto.offset.reset': 'earliest',
    'enable.auto.commit': False,
}

consumer = Consumer(consumer_config)
consumer.subscribe(['orders'])

try:
    while True:
        msg = consumer.poll(timeout=1.0)
        if msg is None:
            continue
        if msg.error():
            if msg.error().code() == KafkaError._PARTITION_EOF:
                continue
            print(f"Error: {msg.error()}")
            break

        event = json.loads(msg.value().decode("utf-8"))
        print(f"Processing: {event}")

        # Manual commit after successful processing
        consumer.commit(asynchronous=False)
finally:
    consumer.close()
```

### Java (standard Kafka client)

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.*;
import java.util.*;
import java.time.Duration;

public class RedpandaExample {

    public static void main(String[] args) {
        // Producer — note: connecting to Redpanda, no code change needed
        Properties producerProps = new Properties();
        producerProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,
            "localhost:9092");
        producerProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
            StringSerializer.class.getName());
        producerProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
            StringSerializer.class.getName());
        producerProps.put(ProducerConfig.ACKS_CONFIG, "all");
        producerProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

        try (KafkaProducer<String, String> producer =
                new KafkaProducer<>(producerProps)) {

            ProducerRecord<String, String> record = new ProducerRecord<>(
                "orders", "ord-7829",
                "{\"type\":\"OrderPlaced\",\"orderId\":\"ord-7829\"}");

            producer.send(record, (metadata, exception) -> {
                if (exception != null) {
                    System.err.println("Send failed: " + exception);
                } else {
                    System.out.printf("Sent to %s[%d]@%d%n",
                        metadata.topic(),
                        metadata.partition(),
                        metadata.offset());
                }
            });
        }

        // Consumer — also unchanged
        Properties consumerProps = new Properties();
        consumerProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,
            "localhost:9092");
        consumerProps.put(ConsumerConfig.GROUP_ID_CONFIG,
            "payment-service");
        consumerProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
            StringDeserializer.class.getName());
        consumerProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
            StringDeserializer.class.getName());
        consumerProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG,
            "earliest");

        try (KafkaConsumer<String, String> consumer =
                new KafkaConsumer<>(consumerProps)) {

            consumer.subscribe(List.of("orders"));

            while (true) {
                ConsumerRecords<String, String> records =
                    consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("Received: key=%s value=%s "
                        + "partition=%d offset=%d%n",
                        record.key(), record.value(),
                        record.partition(), record.offset());
                }

                consumer.commitSync();
            }
        }
    }
}
```

### Go (franz-go)

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/twmb/franz-go/pkg/kgo"
)

func main() {
	// Producer
	client, err := kgo.NewClient(
		kgo.SeedBrokers("localhost:9092"), // Redpanda
		kgo.DefaultProduceTopic("orders"),
		kgo.RequiredAcks(kgo.AllISRAcks()),
	)
	if err != nil {
		log.Fatal(err)
	}
	defer client.Close()

	record := &kgo.Record{
		Key:   []byte("ord-7829"),
		Value: []byte(`{"type":"OrderPlaced","orderId":"ord-7829"}`),
	}

	results := client.ProduceSync(context.Background(), record)
	for _, pr := range results {
		if pr.Err != nil {
			log.Printf("Produce error: %v", pr.Err)
		} else {
			fmt.Printf("Produced to %s[%d]@%d\n",
				pr.Record.Topic,
				pr.Record.Partition,
				pr.Record.Offset)
		}
	}

	// Consumer
	consumerClient, err := kgo.NewClient(
		kgo.SeedBrokers("localhost:9092"),
		kgo.ConsumeTopics("orders"),
		kgo.ConsumerGroup("payment-service"),
	)
	if err != nil {
		log.Fatal(err)
	}
	defer consumerClient.Close()

	for {
		fetches := consumerClient.PollFetches(context.Background())
		fetches.EachRecord(func(r *kgo.Record) {
			fmt.Printf("Consumed: key=%s value=%s partition=%d offset=%d\n",
				string(r.Key), string(r.Value),
				r.Partition, r.Offset)
		})
		consumerClient.AllowRebalance()
	}
}
```

The point of these examples is not the code — it is that the code is *identical* to what you would write for Kafka. The only difference is the bootstrap server address. That is the entire value proposition of Kafka API compatibility, demonstrated in three languages.

---

## Verdict

Redpanda is the most credible Kafka alternative on the market. It delivers on its core promises: lower latency, simpler operations, and genuine Kafka compatibility. The single-binary deployment, self-tuning behaviour, and elimination of ZooKeeper are not just marketing talking points — they represent a meaningful reduction in operational burden.

The question is not whether Redpanda is good. It is. The question is whether it is good enough to justify switching from Kafka, or compelling enough to choose over Kafka for a new project.

For new projects: Redpanda is a strong default choice, particularly for smaller teams. The operational simplicity alone may justify the slightly smaller ecosystem. If you do not need Kafka Streams or ksqlDB, and you are not betting on Confluent's specific enterprise features, Redpanda gives you the same programming model with less operational overhead.

For existing Kafka deployments: migrate if you have a concrete problem that Redpanda solves — operational complexity that is consuming your team, latency requirements that Kafka cannot meet, cost pressure from your node count. Do not migrate because Redpanda is newer or because C++ sounds faster. Migration has costs, and "it might be better" is not a business case.

For enterprises with compliance requirements: evaluate the enterprise license carefully. The BSL licensing model means the community edition has restrictions on offering Redpanda as a managed service (which affects cloud providers, not end users), but enterprise features like audit logging and advanced RBAC may require a commercial agreement.

Redpanda has done something rare: it has built a genuinely competitive alternative to a dominant platform without requiring users to learn a new API. That alone deserves respect. Whether it becomes the default choice for stream processing or remains a compelling alternative depends on execution, ecosystem growth, and whether "Kafka without the JVM" is a big enough differentiator to overcome Kafka's massive installed base.

For now, Redpanda is the answer to "I wish Kafka were simpler to run." If that is your wish, your wish has been granted. Check the fine print, as always.
