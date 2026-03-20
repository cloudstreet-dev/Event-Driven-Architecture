# Google Pub/Sub and Azure Event Hubs

Not every organisation runs on AWS, and not every organisation should. Google Cloud Platform and Microsoft Azure each built their own managed event infrastructure, and while neither commands the same market share as AWS, both have engineering depth that deserves serious examination rather than the "also-ran" treatment they sometimes get in broker comparisons.

This chapter covers two services that solve similar problems in meaningfully different ways. Google Pub/Sub is a topic-and-subscription messaging service with a focus on simplicity and horizontal scaling. Azure Event Hubs is a partitioned log service with a focus on high-throughput ingestion and Kafka wire protocol compatibility. They are not interchangeable, and understanding where each shines — and where each quietly falls apart — will save you from making an expensive mistake.

---

# Google Cloud Pub/Sub

## Overview

Google Cloud Pub/Sub launched in 2015, though its roots go back much further into Google's internal messaging infrastructure. If you have read the original Millwheel paper (2013), you have seen the ancestry. Pub/Sub was designed to be a globally distributed, fully managed messaging service that "just works" — and to a surprising degree, it does.

Pub/Sub is built on the same infrastructure that handles messaging inside Google. This is not mere marketing. The service runs on Forstore, Google's distributed messaging backbone, which means it inherits properties like global message routing, synchronous replication across zones, and throughput that scales without any capacity planning from the user. You create a topic, publish messages, and Google handles the rest. The simplicity is the product.

## Architecture

Pub/Sub's architecture revolves around two primitives: **topics** and **subscriptions**.

A **topic** is a named resource to which publishers send messages. Unlike Kafka topics (which are partitioned logs), Pub/Sub topics are logical channels. There is no partition concept visible to the user. Google handles sharding internally, which means you never think about partition counts, key distribution, or rebalancing. This is either a feature or a limitation, depending on how much control you want.

A **subscription** represents a subscriber's interest in a topic. Crucially, a single topic can have multiple subscriptions, and each subscription receives an independent copy of every message. This is Pub/Sub's fan-out mechanism. Within a subscription, messages are delivered to consumer instances in a load-balanced fashion.

**Pull delivery** is the default model. Consumers call `pull` (or the more efficient `StreamingPull` which maintains a long-lived bidirectional gRPC connection) to receive messages. After processing, the consumer sends an `acknowledge` (ack). Unacknowledged messages are redelivered after the `ackDeadline` (configurable from 10 seconds to 600 seconds). This is conceptually similar to SQS's visibility timeout.

**Push delivery** flips the model. Pub/Sub sends messages to a configured HTTPS endpoint. The endpoint returns a 2xx status code to acknowledge receipt. Push subscriptions are useful when your consumer is a Cloud Function, Cloud Run service, or any HTTP endpoint that can handle webhooks. The push subscriber does not need to maintain a connection or poll.

**Ordering keys** provide within-key ordering guarantees. When you publish messages with the same ordering key, Pub/Sub guarantees that a single subscriber receives them in order. Without ordering keys, message ordering is best-effort. This is the closest equivalent to Kafka's partition-key ordering, but the implementation is different: ordering is per-subscription and per-key, and enabling ordering on a subscription caps throughput for a given key at about 1 MB/s. Publish with ordering keys only when you need ordering. Do not use them "just in case."

**Exactly-once delivery** was added in 2022 and applies within a subscription. When enabled, Pub/Sub guarantees that an acknowledged message will not be delivered again, within the bounds of the ack deadline. This is implemented through server-side deduplication of acks, not deduplication of publishes. The distinction matters: the publisher can still retry a publish and create duplicates; it is the *delivery to the subscriber* that is deduplicated. To get true end-to-end exactly-once, your publisher needs its own deduplication (or you use idempotent processing, which you should be doing anyway).

**Dead lettering** routes messages that have been delivered but not acknowledged beyond a configurable `maxDeliveryAttempts` to a dead letter topic. The dead letter topic is itself a Pub/Sub topic with its own subscriptions, so you can process dead-lettered messages however you like. This works well.

**Message retention** defaults to 7 days but can be set up to 31 days. Acknowledged messages can also be retained (for replay purposes) for up to 31 days. Unacknowledged messages that exceed the retention period are dropped.

**Seek and replay**: You can seek a subscription to a timestamp, which redelivers all messages from that point forward. You can also seek to a snapshot, which captures the acknowledgement state of a subscription at a point in time. This gives you a limited form of replay — not as powerful as Kafka's offset-based replay, but significantly more than what SQS offers (which is nothing).

## Strengths

**Radical simplicity.** No partitions to manage. No rebalancing. No broker sizing. You create a topic, create subscriptions, publish messages. The operational surface area is so small that there is almost nothing to get wrong at the infrastructure level. This is a genuine competitive advantage over self-managed alternatives.

**Auto-scaling that actually works.** Pub/Sub scales from zero to millions of messages per second without any configuration changes. You do not pre-provision throughput. You do not monitor shard utilization. Google's internal infrastructure handles scaling transparently. For bursty workloads, this is invaluable.

**Global availability.** Pub/Sub is a global service — topics and subscriptions are not region-scoped (though you can configure message storage policies to restrict data residency). Messages are replicated synchronously across zones within a region. This gives you multi-zone durability without any additional configuration.

**Dataflow integration.** Google Cloud Dataflow (the managed Apache Beam runner) integrates deeply with Pub/Sub. The Pub/Sub I/O connector handles watermarking, windowing, and exactly-once processing out of the box. If your event processing pipeline involves stream processing (windowed aggregations, sessionization, complex event processing), the Pub/Sub + Dataflow combination is one of the smoothest paths available.

**Reasonable pricing at moderate scale.** Pub/Sub charges $40 per TiB of data published and delivered. For moderate throughput with small messages, this is competitive with self-hosted alternatives when you factor in operational costs.

## Weaknesses

**Cost at high scale.** That $40/TiB pricing adds up. If you are ingesting 1 TB of messages per day, your monthly Pub/Sub bill for data alone is roughly $1,200 for publishing plus $1,200 for each subscription that receives the data. With three subscriptions, you are at $4,800/month just for data movement. A Kafka cluster handling the same throughput on GKE would cost less, though the comparison is unfair until you add the cost of the engineer who operates it.

**Ordering limitations.** Ordering keys limit throughput to about 1 MB/s per key. If you need strict ordering on a high-throughput stream, you need to shard your ordering keys carefully. If you need total ordering across all messages in a topic, Pub/Sub cannot give you that. Kafka can (within a single partition), though it comes with its own trade-offs.

**No partition-level control.** The lack of user-visible partitions is a strength for simplicity but a weakness for use cases where you want partition-level assignment, consumer-partition affinity, or compaction. Pub/Sub does not support log compaction — if you need the "latest value per key" pattern, Pub/Sub is the wrong tool.

**GCP lock-in.** Pub/Sub's API is proprietary. There is an open-source emulator for local development, and the Pub/Sub Lite product offered cost optimization (though it was deprecated in 2024 in favour of standard Pub/Sub). But if you move off GCP, your Pub/Sub code needs a full rewrite.

**Eventual delivery.** Pub/Sub does not guarantee delivery latency. Under normal conditions, latency is tens of milliseconds. Under load or during internal rebalancing, it can spike. The SLA guarantees availability, not latency. For latency-sensitive applications, this indeterminism can be a problem.

**No native stream processing.** Pub/Sub is a messaging layer, not a stream processing engine. For windowed aggregations, joins, or complex event processing, you need Dataflow, Flink, or application-level code. This is a design choice, not a flaw — but Kafka Streams and ksqlDB spoil you into thinking your broker should do everything.

## Ideal Use Cases

- **Event-driven microservices on GCP** where simplicity and managed scaling are priorities.
- **Bursty ingest workloads** (IoT telemetry, user clickstreams, log aggregation) where pre-provisioning capacity is impractical.
- **Pub/Sub + Dataflow pipelines** for real-time stream processing.
- **Global event distribution** where multi-region publishing is a requirement.
- **Teams with limited infrastructure expertise** who need a messaging layer that requires near-zero operational investment.

## Operational Reality

Running Pub/Sub is delightfully boring. You monitor subscription backlog (`num_undelivered_messages`) and oldest unacked message age (`oldest_unacked_message_age`). When backlog grows, you scale your consumers. When message age grows, you investigate why consumers are slow. That is essentially the entire operational story.

Monitoring is done through Cloud Monitoring (formerly Stackdriver). The built-in metrics are adequate for most use cases. Alerting on `oldest_unacked_message_age` is the most important operational practice — if this number climbs, your consumers are falling behind, and eventually messages will exceed the retention period and be lost.

**Cost surprises** usually come from three places: (1) multiple subscriptions multiplying data delivery costs, (2) retained acknowledged messages accumulating storage charges, and (3) message attribute sizes counting toward data volume. A message with a 100-byte body and 400 bytes of attributes is billed for 500 bytes.

## Code Examples (Python)

```python
from google.cloud import pubsub_v1
from concurrent.futures import TimeoutError
import json

project_id = "my-project"
topic_id = "order-events"
subscription_id = "order-processor"

# --- Publisher ---
publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path(project_id, topic_id)

def publish_order_event(order_id: str, total: float, region: str):
    """Publish with ordering key for per-order ordering."""
    data = json.dumps({
        "orderId": order_id,
        "totalAmount": total,
        "region": region,
    }).encode("utf-8")

    # ordering_key ensures all events for the same order
    # are delivered in order to a single subscriber
    future = publisher.publish(
        topic_path,
        data,
        ordering_key=order_id,
        event_type="OrderPlaced",  # custom attribute for filtering
    )
    message_id = future.result()
    print(f"Published {message_id} for order {order_id}")


# --- Subscriber (streaming pull) ---
subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path(project_id, subscription_id)

def callback(message: pubsub_v1.subscriber.message.Message):
    """Process a single message. Ack on success, nack on failure."""
    try:
        event = json.loads(message.data.decode("utf-8"))
        event_type = message.attributes.get("event_type", "unknown")

        print(f"Received {event_type}: order {event['orderId']}")
        process_order(event)

        message.ack()
    except Exception as e:
        print(f"Processing failed: {e}")
        message.nack()  # Message will be redelivered after ack deadline


def consume():
    """Start streaming pull subscriber. Blocks until interrupted."""
    streaming_pull_future = subscriber.subscribe(
        subscription_path,
        callback=callback,
        flow_control=pubsub_v1.types.FlowControl(
            max_messages=100,          # Max outstanding messages
            max_bytes=10 * 1024 * 1024  # Max outstanding bytes (10 MB)
        ),
    )
    print(f"Listening on {subscription_path}...")

    try:
        streaming_pull_future.result()  # Blocks forever
    except TimeoutError:
        streaming_pull_future.cancel()
        streaming_pull_future.result()  # Wait for cleanup


def process_order(event: dict):
    pass
```

## Verdict: Google Pub/Sub

Pub/Sub is the messaging service for teams who want to think about their application, not their infrastructure. Its simplicity-to-capability ratio is the best in the cloud-native messaging space. You trade fine-grained control (no partitions, no compaction, limited ordering control) for a service that scales effortlessly and requires almost no operational investment.

If you are on GCP, Pub/Sub is the default choice for event-driven architecture, and you need a specific reason *not* to use it. If you are choosing a cloud, Pub/Sub alone is not a reason to choose GCP — but if you are already there, it is one of the better reasons to stay.

---

# Azure Event Hubs

## Overview

Azure Event Hubs launched in 2014 as Microsoft's answer to high-throughput event ingestion. Where Google built a messaging service that hides its internals behind simplicity, Microsoft built a partitioned log service that wears its architecture on its sleeve. If you squint, Event Hubs looks a lot like Kafka. This is not a coincidence — the partitioned append-only log model is the same, and since 2018, Event Hubs has supported the Kafka wire protocol directly.

Event Hubs is positioned as the entry point for big data pipelines on Azure. It sits in front of Azure Stream Analytics, Azure Functions, Azure Data Lake, and the rest of Microsoft's data ecosystem. It is also a legitimate Kafka replacement for teams that want the Kafka programming model without operating Kafka infrastructure.

## Architecture

Event Hubs is organised around a hierarchy of concepts that will be familiar if you have used Kafka.

**Namespaces** are the top-level container. A namespace maps to a cluster of brokers and defines the pricing tier (Basic, Standard, Premium, or Dedicated). Think of it as a Kafka cluster equivalent.

**Event Hubs** (confusingly, the resource shares its name with the service) are the equivalent of Kafka topics. Each event hub has a configurable number of partitions (2–32 on Standard, up to 2,000 on Dedicated).

**Partitions** are the unit of parallelism and ordering. Events within a partition are strictly ordered and appended to an immutable log. Producers can specify a partition key (hashed to a partition) or publish to a specific partition. This is the Kafka model, nearly 1:1.

**Consumer groups** define independent views of the partition log. Each consumer group maintains its own offset per partition. Events are not deleted after consumption — they are retained for a configurable period (1–90 days on Standard and Premium, up to 90 days on Dedicated). This is Kafka's consumer group model.

**Capture** is Event Hubs' killer feature for data pipeline use cases. It automatically writes events to Azure Blob Storage or Azure Data Lake Store in Avro format, at configurable time and size intervals. No consumer code needed. The events just show up in your data lake, partitioned by time. This is operationally lovely — you get a durable archive of every event with zero application code.

**Kafka protocol compatibility** (available on Standard tier and above) means you can point a Kafka client at Event Hubs and it works. Your existing Kafka producers, consumers, Kafka Connect connectors, and even Kafka Streams applications can target Event Hubs with configuration changes only. The compatibility is not 100% — some admin APIs are not supported, consumer group management has some differences, and compacted topics are not available — but for the core produce/consume workflow, it works well enough that teams have migrated off self-hosted Kafka to Event Hubs with minimal code changes.

**Event Hubs Premium** (added in 2021) provides dedicated compute resources within a shared infrastructure. It supports up to 100 partitions per event hub, offers better isolation than Standard tier, and has higher throughput and message size limits (1 MB vs 256 KB on Standard). Premium is positioned as "most of the benefits of Dedicated, without the commitment."

**Event Hubs Dedicated** gives you an entire cluster. You can have up to 2,000 partitions per event hub, retention up to 90 days, and throughput limited only by the cluster capacity (which you control). Dedicated is for organisations processing millions of events per second who need predictable performance and complete isolation.

## Strengths

**Kafka wire protocol compatibility.** This is the headline feature, and it is genuinely useful. Teams with existing Kafka expertise and codebases can migrate to a managed service without rewriting clients. Kafka Connect works. Kafka Streams works (with caveats). The learning curve for Kafka engineers moving to Event Hubs is measured in hours, not weeks.

**Capture to blob storage.** Automatic archival to Azure Blob Storage in Avro format is effortlessly useful. No Lambda functions, no custom consumers, no cron jobs. Data lands in your data lake, properly partitioned, ready for batch processing with Spark, Databricks, or Synapse Analytics. For organisations that need both real-time processing and batch analytics on the same event streams, Capture closes a gap that Kafka requires third-party tools (Kafka Connect, S3 sink connector) to fill.

**Azure ecosystem integration.** Event Hubs feeds directly into Azure Functions (with scaling based on partition count), Stream Analytics (for SQL-like real-time queries), Azure Data Explorer (for log analytics), and Synapse Analytics (for big data processing). If your organisation is on Azure, the integration plumbing is already built.

**Managed scaling on Premium/Dedicated tiers.** Premium tier auto-scales processing units based on load. You do not manage broker instances or worry about disk capacity. On Dedicated, you manage cluster capacity but not individual brokers.

**Long retention.** Up to 90 days on Premium and Dedicated tiers. This is longer than most managed messaging services (Pub/Sub maxes at 31 days, SQS at 14 days) and makes Event Hubs viable for replay-heavy workloads.

## Weaknesses

**Partition limit on Standard tier.** Standard tier limits you to 32 partitions per event hub. This caps your consumer parallelism and throughput. For high-throughput workloads, you need Premium (100 partitions) or Dedicated (2,000 partitions), which are significantly more expensive.

**No log compaction.** This is the most significant gap in the Kafka compatibility story. Kafka's compacted topics are foundational for the "event store as a database" pattern and for maintaining latest-state-per-key tables. Event Hubs simply does not support it. If you need compaction, you need actual Kafka (or Redpanda, or you build your own compaction pipeline to a database, which is ugly).

**Azure lock-in.** The Kafka protocol compatibility mitigates this somewhat — your client code is portable. But the Capture, monitoring, IAM, and networking integrations are Azure-specific. Moving from Event Hubs Capture to S3 requires new infrastructure.

**Pricing complexity.** Event Hubs pricing involves throughput units (Standard), processing units (Premium), or capacity units (Dedicated), plus per-million-events ingress charges, plus storage costs, plus Capture storage costs. Comparing the total cost to alternatives requires a spreadsheet. This is not unique to Event Hubs — Azure pricing is consistently the most opaque of the three major clouds — but it makes cost estimation harder than it should be.

**Consumer group limit.** Standard tier allows 20 consumer groups per event hub. Premium allows 100. This is generous for most use cases but can be limiting for organisations that use consumer groups heavily (one per microservice, one per analytics pipeline, one per testing environment, etc.).

**Throughput units on Standard tier.** Each throughput unit provides 1 MB/s ingress and 2 MB/s egress. You can have up to 40 throughput units with auto-inflate enabled. If you need more, you are on Premium or Dedicated. The throughput unit model requires capacity planning, which partially undermines the "managed service" value proposition.

## Ideal Use Cases

- **High-throughput event ingestion** for IoT, telemetry, clickstream, and log aggregation on Azure.
- **Kafka migration to managed infrastructure** where teams want to retain Kafka clients and patterns without operating Kafka clusters.
- **Data pipeline ingestion** where Capture provides automatic archival to the data lake.
- **Real-time + batch analytics** where the same event stream feeds both Stream Analytics (real-time) and Synapse (batch).
- **Organisations with existing Azure investment** where the ecosystem integration reduces glue code.

## Operational Reality

Operating Event Hubs on Standard tier involves monitoring throughput unit utilization and partition health. Enable auto-inflate for throughput units to handle bursts. Monitor incoming/outgoing messages, throttled requests, and consumer lag.

On Premium and Dedicated tiers, the operational burden is lower — auto-scaling handles throughput management, and you focus on partition distribution and consumer health.

**Monitoring** uses Azure Monitor metrics. Key metrics: `IncomingMessages`, `OutgoingMessages`, `ThrottledRequests` (your canary for capacity issues), `IncomingBytes`/`OutgoingBytes`, and consumer lag (available through the Kafka consumer group protocol or the Event Hubs SDK's `PartitionProperties.LastEnqueuedSequenceNumber` minus your current offset).

**Capture management** is operationally simple — you configure the destination container, the time window (1–15 minutes), and the size window (10–500 MB), and Event Hubs handles the rest. Monitor for capture failures and be aware that small time windows with low throughput will produce many small Avro files, which is suboptimal for downstream Spark processing. A common pattern is to set a 5-minute capture window and run a periodic compaction job on the resulting files.

**Partition management** is the main planning exercise. Unlike Pub/Sub, partition count matters and affects performance. You cannot decrease partition count after creation (only increase it on Premium/Dedicated). Choosing the right initial partition count requires estimating your peak throughput and consumer parallelism.

## Code Examples (Python)

```python
from azure.eventhub import EventHubProducerClient, EventData, EventHubConsumerClient
from azure.eventhub.extensions.checkpointstoreblob import BlobCheckpointStore
import json

CONNECTION_STR = "Endpoint=sb://my-namespace.servicebus.windows.net/;..."
EVENTHUB_NAME = "order-events"

# --- Producer ---
def publish_order_events(events: list[dict]):
    """
    Publish events with partition keys for ordering.
    Uses batching to maximize throughput.
    """
    producer = EventHubProducerClient.from_connection_string(
        conn_str=CONNECTION_STR,
        eventhub_name=EVENTHUB_NAME,
    )

    with producer:
        # Create a batch scoped to a partition key
        # All events in this batch go to the same partition
        for event in events:
            event_data_batch = producer.create_batch(
                partition_key=event["orderId"]
            )
            event_data_batch.add(EventData(json.dumps(event)))
            producer.send_batch(event_data_batch)
            print(f"Sent event for order {event['orderId']}")


# --- Consumer with checkpointing ---
STORAGE_CONNECTION_STR = "DefaultEndpointsProtocol=https;..."
BLOB_CONTAINER_NAME = "eventhub-checkpoints"

def on_event(partition_context, event):
    """Process a single event and checkpoint."""
    body = event.body_as_str()
    order_event = json.loads(body)

    print(f"Partition {partition_context.partition_id}: "
          f"order {order_event['orderId']}")

    process_order(order_event)

    # Checkpoint after processing — stores offset in blob storage
    partition_context.update_checkpoint(event)


def on_error(partition_context, error):
    """Handle errors during event processing."""
    if partition_context:
        print(f"Error on partition {partition_context.partition_id}: {error}")
    else:
        print(f"Error: {error}")


def consume():
    """
    Start consuming from all partitions.
    BlobCheckpointStore persists consumer offsets in Azure Blob Storage,
    enabling consumer restarts without reprocessing.
    """
    checkpoint_store = BlobCheckpointStore.from_connection_string(
        STORAGE_CONNECTION_STR,
        BLOB_CONTAINER_NAME,
    )

    consumer = EventHubConsumerClient.from_connection_string(
        conn_str=CONNECTION_STR,
        consumer_group="$Default",
        eventhub_name=EVENTHUB_NAME,
        checkpoint_store=checkpoint_store,
    )

    with consumer:
        consumer.receive(
            on_event=on_event,
            on_error=on_error,
            starting_position="-1",  # Start from beginning
        )


# --- Using Kafka protocol instead ---
# pip install confluent-kafka
from confluent_kafka import Producer as KafkaProducer, Consumer as KafkaConsumer

def kafka_producer_example():
    """
    Same Event Hubs, but using the Kafka wire protocol.
    Your existing Kafka code works with a config change.
    """
    config = {
        'bootstrap.servers': 'my-namespace.servicebus.windows.net:9093',
        'security.protocol': 'SASL_SSL',
        'sasl.mechanism': 'PLAIN',
        'sasl.username': '$ConnectionString',
        'sasl.password': CONNECTION_STR,
    }

    producer = KafkaProducer(config)
    producer.produce(
        topic=EVENTHUB_NAME,
        key=b"ord-7829",
        value=json.dumps({"orderId": "ord-7829", "total": 149.99}).encode(),
    )
    producer.flush()


def process_order(event: dict):
    pass
```

## Verdict: Azure Event Hubs

Event Hubs is the right answer for organisations that are committed to Azure and need high-throughput event ingestion. The Kafka protocol compatibility is its trump card — it gives you the Kafka programming model with managed infrastructure, which is a genuine value proposition for teams that know Kafka but do not want to run it.

Capture is the other standout feature. Automatic archival to blob storage, in a structured format, with no application code, solves a problem that every event-driven system eventually faces. If your architecture involves both real-time processing and batch analytics on the same event data, Event Hubs + Capture is one of the cleanest solutions available.

The limitations are real but predictable. No compaction means Event Hubs cannot fully replace Kafka for all use cases. The partition and consumer group limits on Standard tier push high-throughput workloads to more expensive tiers. And Azure's pricing model rewards patience and a good spreadsheet.

---

# Comparing the Cloud-Native Options

Here is a direct comparison of all three cloud-native options covered in this chapter and the previous one:

| Dimension | AWS SNS+SQS / EventBridge | Google Cloud Pub/Sub | Azure Event Hubs |
|-----------|--------------------------|---------------------|-----------------|
| **Model** | Queue + pub/sub + event bus | Topic + subscription | Partitioned log |
| **Ordering** | FIFO queues (300 TPS) | Ordering keys (~1 MB/s per key) | Per-partition (Kafka model) |
| **Replay** | EventBridge archive (limited) | Seek to timestamp (31 days) | Offset-based (up to 90 days) |
| **Compaction** | No | No | No |
| **Kafka compat** | No (use MSK) | No | Yes (wire protocol) |
| **Auto-scaling** | Fully automatic | Fully automatic | Throughput units / auto-inflate |
| **Max retention** | 14 days (SQS) / configurable (EB) | 31 days | 90 days (Premium/Dedicated) |
| **Max message size** | 256 KB | 10 MB | 1 MB (Premium) / 256 KB (Standard) |
| **Dead lettering** | SQS DLQ | Dead letter topic | No native DLQ (handle in consumer) |
| **Ecosystem** | Deepest AWS integration | GCP + Dataflow | Azure + Kafka ecosystem |
| **Pricing model** | Per-request (SQS) / per-event (EB) | Per-data-volume | Per-throughput-unit + per-event |
| **Ops burden** | Near zero | Near zero | Low (Standard) to near zero (Premium) |

### When to Choose Which

**Choose AWS SNS+SQS/EventBridge** when you are on AWS, want zero operational overhead, and your throughput requirements are moderate. The EventBridge rule engine is the most sophisticated content-based router among the three.

**Choose Google Pub/Sub** when you are on GCP, want the simplest possible mental model, or your workloads are bursty and unpredictable. Pub/Sub's auto-scaling is the most transparent of the three — you truly never think about capacity.

**Choose Azure Event Hubs** when you are on Azure, need high-throughput ordered event streams, want Kafka compatibility without operating Kafka, or need long retention with Capture for data lake integration.

**Choose none of them** when you need sub-millisecond latency, log compaction, cross-cloud portability, or you have the team and mandate to operate your own infrastructure. In those cases, look at Kafka, Redpanda, or NATS.

---

## Final Verdict

Every cloud provider's managed messaging service is good enough for most workloads. That is a deliberately boring statement, and it is true. The differences between them matter at the margins — ordering guarantees, pricing models, ecosystem integration, replay capabilities — but the core proposition is the same: you get a durable, scalable messaging layer without operating it yourself.

The most important factor in choosing between them is *which cloud you are already on*. If you are on GCP, use Pub/Sub. If you are on Azure, use Event Hubs. If you are on AWS, use SNS/SQS/EventBridge. Cross-cloud messaging is a solvable problem, but it is not a problem you want to solve unless you genuinely have multi-cloud workloads.

Do not let the choice of messaging service drive your choice of cloud provider. That tail is far too small to wag that dog.
