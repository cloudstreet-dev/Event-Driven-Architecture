# Amazon SNS/SQS and EventBridge

Every cloud provider eventually builds a message broker. Amazon built three of them, gave them names that sound like government agencies, and then told you to use all of them together. Welcome to the AWS-native path to event-driven architecture, where the infrastructure is invisible, the scaling is automatic, and the bill is... educational.

This chapter covers Simple Notification Service (SNS), Simple Queue Service (SQS), and EventBridge — the trio that forms the backbone of serverless event-driven design on AWS. Each solves a different problem. Together, they form something surprisingly coherent, provided you can navigate the configuration surface area. We will also touch on EventBridge Scheduler, because AWS apparently believes no service is complete until it can trigger a Lambda on a cron expression.

---

## Overview

### SNS: The Town Crier

Amazon Simple Notification Service launched in 2010, making it one of the oldest managed pub/sub services still in active use. Its job is straightforward: you publish a message to a topic, and SNS delivers it to every subscriber. Fan-out. That is the pitch, and it delivers on it reliably.

SNS supports multiple subscriber types — SQS queues, Lambda functions, HTTP/S endpoints, email, SMS, and mobile push notifications. This makes it the Swiss Army knife of notification delivery, though like most Swiss Army knives, some of the tools are more useful than others. (The email subscriber, for instance, is fine for ops alerts and catastrophically wrong for anything customer-facing.)

### SQS: The Patient Queue

SQS is older than SNS — it launched in 2004, making it one of AWS's first services, period. It is a fully managed message queue. You put messages in, you take messages out, and in between SQS handles durability, scaling, and the existential dread of distributed systems. It was one of the services described in the famous Werner Vogels "everything fails all the time" era, and its design reflects that philosophy: simple, durable, relentlessly boring.

### EventBridge: The Smart Router

EventBridge arrived in 2019, born from the ashes of CloudWatch Events (which still exists underneath, a fact that will confuse you exactly once during a debugging session). EventBridge is a serverless event bus with content-based routing. Where SNS says "here is a message, deliver it to everyone who subscribed to this topic," EventBridge says "here is an event, deliver it to everyone whose *rule* matches its *content*." This is a meaningful distinction. EventBridge is the opinionated one.

EventBridge also integrates with over 90 AWS services as event sources and over 20 as targets. It has a schema registry. It has archive and replay. It has Pipes for point-to-point integrations. AWS has made it clear that EventBridge is the future of event-driven architecture on their platform, and they are investing accordingly.

---

## Architecture

### SNS Internals

SNS is a push-based system. When you publish a message to a topic, SNS fans it out to all subscribers in parallel. There is no consumer polling, no long-lived connections. SNS pushes and moves on.

**Topic types:**

- **Standard topics**: Best-effort ordering, at-least-once delivery, nearly unlimited throughput (the published limit is 100,000 messages per second per topic for API calls, though the actual delivery fan-out is higher). Messages may be delivered out of order and may be delivered more than once.
- **FIFO topics**: Strict ordering within a message group, exactly-once delivery (within the deduplication window), but throughput is capped at 300 publishes per second (or 10 MB/s), and subscribers are limited to SQS FIFO queues. The trade-off is exactly what you would expect: you get ordering guarantees in exchange for throughput and flexibility.

**Message filtering** is one of SNS's underappreciated features. Instead of creating a topic per event type (a pattern that scales poorly), you can publish all events to a single topic and attach filter policies to subscriptions. Filters can match on message attributes using exact values, prefix matching, numeric ranges, and even exists/not-exists checks. This moves routing logic from your application code into infrastructure configuration, which is either a win for separation of concerns or a debugging nightmare, depending on how well you document your filter policies.

**Delivery policies** control retry behaviour for HTTP/S subscribers. You can configure the number of retries, backoff functions (linear, geometric, exponential), and the time between retries. For SQS and Lambda subscribers, delivery is effectively guaranteed by the underlying integration — SNS will retry until SQS accepts the message.

### SQS Internals

SQS is a pull-based system. Messages sit in a queue until a consumer fetches them. This is the fundamental architectural difference from SNS: SQS is about *buffering*, not *broadcasting*.

**Queue types:**

- **Standard queues**: Nearly unlimited throughput, at-least-once delivery, best-effort ordering. Messages are stored redundantly across multiple availability zones. The "best-effort ordering" part means that messages *usually* come out in roughly the order they went in, but you must not depend on this. If you are depending on this, you will discover your mistake at 2 AM on a Saturday when a partition rebalance shuffles your messages.
- **FIFO queues**: Exactly-once processing (via deduplication), strict ordering within message groups, but throughput is limited to 300 messages per second without batching (3,000 with batching and high-throughput mode). Message groups are the key concept — ordering is guaranteed *within* a group, not across groups, which lets you partition your ordering requirements and get some parallelism back.

**Visibility timeout** is SQS's answer to the question "what happens if a consumer takes a message and then dies?" When a consumer receives a message, SQS hides it from other consumers for a configurable period (default 30 seconds, maximum 12 hours). If the consumer does not delete the message within that window, it becomes visible again for another consumer to pick up. Get this value wrong and you get either duplicate processing (too short) or long delays after failures (too long). Most teams get it wrong at least once.

**Long polling** is the answer to the question "why is my SQS consumer burning money on empty ReceiveMessage calls?" Instead of returning immediately when no messages are available, long polling waits up to 20 seconds for a message to arrive. This reduces empty responses and costs. There is no good reason not to use it, and yet I have reviewed production systems where it was not enabled.

**Dead letter queues (DLQ)** catch messages that have been received but not successfully processed after a configurable number of attempts (the `maxReceiveCount`). When a message exceeds the receive count, SQS moves it to the DLQ. This is your safety net — without it, poison messages will cycle through your consumer forever, consuming resources and generating alerts. Every production SQS queue should have a DLQ. This is not a suggestion.

**Message deduplication** in FIFO queues uses either a content-based hash (SHA-256 of the message body) or an explicit deduplication ID. The deduplication window is five minutes. If you send the same message twice within five minutes, the second one is silently dropped. After five minutes, all bets are off. This means FIFO queues provide exactly-once delivery *within a window*, not for all time. Plan accordingly.

### The SNS+SQS Fan-Out Pattern

This is the bread and butter of AWS event-driven architecture. You publish to an SNS topic, and the topic delivers to multiple SQS queues. Each queue feeds a different consumer. This gives you:

1. **Fan-out**: One event reaches multiple consumers.
2. **Buffering**: Each consumer processes at its own pace.
3. **Failure isolation**: If one consumer fails, the others are unaffected.
4. **Replay from DLQ**: Failed messages land in per-consumer DLQs.

```
Producer → SNS Topic → SQS Queue A → Consumer A
                     → SQS Queue B → Consumer B
                     → SQS Queue C → Consumer C
```

This pattern is so common that AWS has special integrations for it. SNS can deliver directly to SQS with no intermediate HTTP call, and the IAM policies basically write themselves (or more accurately, the CloudFormation templates do).

The limitation is that SNS+SQS gives you *topic-based* routing. If you need content-based routing — "send this event to Consumer A only if the `region` field is `eu-west-1`" — you either use SNS message filtering or you reach for EventBridge.

### EventBridge Architecture

EventBridge is structured around three core concepts:

**Event buses** are the pipelines. Every AWS account has a default event bus that receives events from AWS services (EC2 state changes, S3 notifications, etc.). You can create custom event buses for your application events. Events are JSON objects with a specific envelope structure:

```json
{
  "version": "0",
  "id": "12345678-1234-1234-1234-123456789012",
  "detail-type": "OrderPlaced",
  "source": "com.mycompany.orders",
  "account": "123456789012",
  "time": "2025-11-14T09:32:17Z",
  "region": "us-east-1",
  "resources": [],
  "detail": {
    "orderId": "ord-7829",
    "totalAmount": 149.99
  }
}
```

**Rules** are the routing logic. Each rule has an event pattern (a JSON filter) and one or more targets. Event patterns support exact matching, prefix matching, numeric ranges, exists/not-exists, and boolean operators. Rules can match on any field in the event envelope or the detail payload. You can have up to 300 rules per event bus (a soft limit that is raiseable but indicates roughly where AWS expects you to hit design problems).

**Schema registry** is EventBridge's attempt to bring some order to the chaos. It can automatically discover schemas from events flowing through your bus, and you can also register schemas manually. This integrates with code generation tools so your consumers can work with typed objects instead of raw JSON. In practice, the auto-discovery is useful for exploration and the manual registry is useful for contracts.

**Archive and replay** lets you store events and replay them later. You can archive all events or filter by pattern. Replay sends the archived events back through the bus, where they match rules just like fresh events. This is genuinely useful for testing, disaster recovery, and populating new consumers with historical data. The limitation is that replay replays through the same rules, so if your rules have changed since the events were archived, you may get different routing behaviour. Also, replay is not instantaneous — large archives take time to replay, and the throughput is not documented.

**EventBridge Pipes** are point-to-point integrations with optional filtering, enrichment, and transformation. A pipe connects a source (SQS, Kinesis, DynamoDB Streams, Kafka, etc.) to a target through an optional filter, enrichment step (Lambda, Step Functions, API Gateway, API destination), and transformation. Pipes are the answer to "I just need to get data from A to B with a bit of transformation" — a use case that previously required writing a Lambda function, which always felt like overkill.

**EventBridge Scheduler** is a managed scheduler that can invoke targets on a schedule (cron or rate) or at a specific time. The "specific time" part is the interesting bit — you can schedule a one-time invocation at a future time, which is useful for delayed processing, reminders, and time-based workflows. It supports up to millions of scheduled invocations, which puts it in a different league from CloudWatch Events rules with cron expressions (which are limited to 300 per account).

---

## Strengths

**Zero operational overhead.** There are no servers to manage, no clusters to resize, no brokers to patch. SNS, SQS, and EventBridge are fully managed. You do not SSH into anything. You do not think about disk space. You do not get paged because a Kafka broker ran out of file descriptors. For teams that want to focus on application logic rather than infrastructure, this is genuinely liberating.

**Deep AWS integration.** Over 90 AWS services emit events to EventBridge natively. SQS integrates as a Lambda event source with automatic scaling of consumer concurrency. Step Functions can wait for events. API Gateway can publish to SNS. The integration surface area is enormous, and it mostly works without custom glue code.

**Pay-per-use pricing.** You pay for what you use, not for what you provision. An SQS queue that processes ten messages a day costs fractions of a cent. This is transformative for development and staging environments, where a Kafka cluster sits idle burning money.

**Managed scaling.** SQS scales to millions of messages per second with no configuration changes. SNS standard topics have no practical throughput ceiling. EventBridge scales to handle event bursts. You do not pre-provision capacity or monitor partition hotspots.

**Durability.** SQS stores messages redundantly across multiple availability zones. Message loss is, for practical purposes, not a concern. The documented durability is not published as a number of nines (unlike S3), but the engineering behind it is serious.

**Security model.** IAM policies, VPC endpoints, encryption at rest (KMS), encryption in transit (TLS). The security tooling integrates with the rest of AWS security. Resource policies on SNS topics and SQS queues enable cross-account access without sharing credentials.

---

## Weaknesses

**Vendor lock-in.** Let us not dance around this. If you build on SNS, SQS, and EventBridge, you are locked into AWS. The APIs are proprietary. The event format (especially EventBridge's envelope) is AWS-specific. Migrating to another cloud or to a self-hosted solution means rewriting your event infrastructure. Some teams accept this trade-off willingly. Some discover it was a trade-off after it is too late to change.

**Throughput limits on FIFO resources.** SQS FIFO queues max out at 300 TPS without batching (3,000 with batching). SNS FIFO topics max at 300 publishes per second. If you need ordering guarantees and high throughput, you will hit these limits faster than you expect. The workaround — sharding across multiple message groups — adds complexity that undermines the "simple" part of SQS.

**Cross-account and cross-region complexity.** Sending events across AWS accounts requires resource policies, IAM roles, and often EventBridge cross-account event bus configurations. It works, but the IAM policy debugging experience is not fun. Cross-region event routing with EventBridge requires setting up rules that forward events to buses in other regions, and each hop adds latency and cost.

**Latency characteristics.** SQS long polling adds up to 20 seconds of latency by design (you can set it lower, but then you pay for more API calls). SNS push delivery is fast but not sub-millisecond. EventBridge rule evaluation adds processing time. If you need single-digit millisecond end-to-end latency, AWS managed services are not your answer. You are looking at Kafka, NATS, or Redis Streams.

**Limited replay.** EventBridge archive and replay exists, but it is not a first-class event log. You cannot randomly access events by offset. Replay is all-or-nothing within a time range. SQS has no replay at all — once a message is deleted, it is gone. If you need event sourcing with full replay capability, you are in Kafka territory, and no amount of AWS marketing can change that.

**Message size limits.** SQS messages max out at 256 KB. SNS messages max at 256 KB. EventBridge events max at 256 KB. If your events are larger, you need the "claim check" pattern (store the payload in S3, put the reference in the message). This is a well-known pattern but it adds complexity, latency, and failure modes.

**Observability gaps.** CloudWatch metrics for SQS give you queue depth, age of oldest message, and approximate message counts. They do not give you consumer lag per consumer, message throughput broken down by message type, or end-to-end latency distributions. You can build these, but you are building them from scratch with custom metrics. After using Kafka's consumer group lag monitoring, SQS feels like flying blind.

---

## Ideal Use Cases

- **Serverless applications** where Lambda is the primary compute. The SQS-Lambda integration is excellent, and the pay-per-use model matches Lambda's pricing.
- **Fan-out patterns** where one event needs to reach multiple independent consumers. SNS+SQS is purpose-built for this.
- **AWS-to-AWS integration** where you need to react to infrastructure events (EC2 state changes, S3 uploads, CodePipeline completions).
- **Low-to-medium throughput workloads** where the simplicity of managed services outweighs the throughput limitations of FIFO ordering.
- **Event routing with complex rules** where EventBridge's content-based filtering saves you from maintaining routing logic in application code.
- **Teams that want to ship features, not operate infrastructure.** This is the real value proposition, and it is genuine.

---

## Operational Reality

Operating SNS, SQS, and EventBridge is, by design, uneventful. There are no clusters to manage, no rebalancing events to monitor, no broker failovers to orchestrate. Your operational concerns shift from "is the broker healthy?" to "is my consumer keeping up?"

**Monitoring** centres on CloudWatch metrics. For SQS: `ApproximateNumberOfMessagesVisible` (queue depth), `ApproximateAgeOfOldestMessage` (consumer lag proxy), and `NumberOfMessagesSent`/`NumberOfMessagesReceived`. For SNS: `NumberOfMessagesPublished` and `NumberOfNotificationsFailed`. For EventBridge: `Invocations`, `FailedInvocations`, and `MatchedEvents`. Set alarms on queue depth and message age. These are your canaries.

**DLQ management** is the operational task that most teams underestimate. Messages land in DLQs for a reason — usually bugs in consumer code, unexpected message formats, or transient downstream failures that outlasted the retry count. You need a process for inspecting DLQ messages, understanding why they failed, fixing the underlying issue, and replaying them. SQS DLQ redrive (which moves messages from the DLQ back to the source queue) was added in 2021 and works well, but "works well" assumes you have fixed whatever caused the failures in the first place.

**Cost management** is straightforward at low scale and surprising at high scale. SQS charges per API call ($0.40 per million requests for standard, $0.50 for FIFO). Each `ReceiveMessage` call is a request, whether or not it returns a message (long polling helps here). Each `SendMessage` is a request. Each `DeleteMessage` is a request. A consumer processing 1,000 messages per second is making roughly 2,000 API calls per second (receive + delete), which is about $4.15 million requests per day, or about $1,660 per month per queue. Scale that to ten queues and you are spending real money. EventBridge charges $1.00 per million events. SNS charges $0.50 per million publishes for standard topics.

**Infrastructure as Code** is non-negotiable. CloudFormation, CDK, Terraform, or Pulumi — pick one and use it. The number of resources involved in a production SNS+SQS setup (topics, queues, DLQs, subscriptions, filter policies, IAM policies, CloudWatch alarms, KMS keys) grows quickly, and managing them through the console is a recipe for configuration drift and 3 AM incidents.

---

## Code Examples

### SNS + SQS: Publishing and Consuming (Python/boto3)

```python
import json
import boto3

sns = boto3.client('sns', region_name='us-east-1')
sqs = boto3.client('sqs', region_name='us-east-1')

# --- Publishing to SNS ---
def publish_order_event(order_id: str, total: float, region: str):
    """Publish an order event to SNS with message attributes for filtering."""
    response = sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789012:order-events',
        Message=json.dumps({
            'orderId': order_id,
            'totalAmount': total,
            'region': region,
            'timestamp': '2025-11-14T09:32:17Z'
        }),
        MessageAttributes={
            'event_type': {
                'DataType': 'String',
                'StringValue': 'OrderPlaced'
            },
            'region': {
                'DataType': 'String',
                'StringValue': region
            }
        }
    )
    print(f"Published message: {response['MessageId']}")
    return response


# --- SNS Subscription with filter policy ---
# This would typically be in your IaC, but for illustration:
def create_filtered_subscription(topic_arn: str, queue_arn: str):
    """Subscribe an SQS queue to SNS with a filter for EU orders only."""
    sns.subscribe(
        TopicArn=topic_arn,
        Protocol='sqs',
        Endpoint=queue_arn,
        Attributes={
            'FilterPolicy': json.dumps({
                'region': ['eu-west-1', 'eu-central-1']
            }),
            'FilterPolicyScope': 'MessageAttributes'
        }
    )


# --- Consuming from SQS ---
def consume_orders(queue_url: str, max_messages: int = 10):
    """
    Long-poll SQS for messages. In production, this runs in a loop
    (or better, use SQS as a Lambda event source).
    """
    response = sqs.receive_message(
        QueueUrl=queue_url,
        MaxNumberOfMessages=max_messages,
        WaitTimeSeconds=20,           # Long polling - always use this
        MessageAttributeNames=['All'],
        AttributeNames=['All']
    )

    messages = response.get('Messages', [])
    for message in messages:
        try:
            # SNS wraps the original message in an envelope
            sns_envelope = json.loads(message['Body'])
            order_event = json.loads(sns_envelope['Message'])

            print(f"Processing order: {order_event['orderId']}")
            process_order(order_event)

            # Delete the message only after successful processing
            sqs.delete_message(
                QueueUrl=queue_url,
                ReceiptHandle=message['ReceiptHandle']
            )
        except Exception as e:
            # Don't delete the message — it will return to the queue
            # after the visibility timeout expires, and eventually
            # land in the DLQ after maxReceiveCount attempts.
            print(f"Failed to process message: {e}")

    return len(messages)


def process_order(event: dict):
    """Your business logic here."""
    pass


# --- SQS FIFO: Sending with ordering ---
def send_fifo_message(queue_url: str, order_id: str, event: dict):
    """
    Send to a FIFO queue with message group ID for ordering.
    All events for the same order are processed in order.
    """
    sqs.send_message(
        QueueUrl=queue_url,  # Must end in .fifo
        MessageBody=json.dumps(event),
        MessageGroupId=order_id,          # Ordering key
        MessageDeduplicationId=f"{order_id}-{event['type']}-{event['version']}"
    )
```

### EventBridge: Publishing Events and Rule Patterns (Python/boto3)

```python
import json
import boto3
from datetime import datetime

events = boto3.client('events', region_name='us-east-1')

# --- Publishing to EventBridge ---
def publish_to_eventbridge(order_id: str, total: float, region: str):
    """Put an event on a custom EventBridge bus."""
    response = events.put_events(
        Entries=[
            {
                'Source': 'com.mycompany.orders',
                'DetailType': 'OrderPlaced',
                'Detail': json.dumps({
                    'orderId': order_id,
                    'totalAmount': total,
                    'region': region,
                    'timestamp': datetime.utcnow().isoformat()
                }),
                'EventBusName': 'orders-bus'
            }
        ]
    )
    # Always check FailedEntryCount — put_events does NOT throw
    # on partial failures. This is a footgun.
    if response['FailedEntryCount'] > 0:
        for entry in response['Entries']:
            if 'ErrorCode' in entry:
                print(f"Failed to publish: {entry['ErrorCode']}: "
                      f"{entry['ErrorMessage']}")
    return response


# --- Batch publishing (up to 10 events per call) ---
def publish_batch(order_events: list[dict]):
    """
    Publish multiple events in a single API call.
    Max 10 entries per put_events call. Max total size 256 KB.
    """
    entries = [
        {
            'Source': 'com.mycompany.orders',
            'DetailType': event['type'],
            'Detail': json.dumps(event['data']),
            'EventBusName': 'orders-bus'
        }
        for event in order_events
    ]

    # put_events accepts max 10 entries, so chunk if needed
    for i in range(0, len(entries), 10):
        chunk = entries[i:i+10]
        response = events.put_events(Entries=chunk)
        if response['FailedEntryCount'] > 0:
            handle_failures(response, chunk)
```

### EventBridge Rule Patterns

```json
// Match all OrderPlaced events from the orders service
{
  "source": ["com.mycompany.orders"],
  "detail-type": ["OrderPlaced"]
}

// Match high-value orders (total > 1000) from EU regions
{
  "source": ["com.mycompany.orders"],
  "detail-type": ["OrderPlaced"],
  "detail": {
    "totalAmount": [{ "numeric": [">", 1000] }],
    "region": [{ "prefix": "eu-" }]
  }
}

// Match any event EXCEPT from the test source
{
  "source": [{ "anything-but": "com.mycompany.test" }]
}

// Match events that have a "priority" field (regardless of value)
{
  "detail": {
    "priority": [{ "exists": true }]
  }
}
```

### EventBridge Scheduler: One-Time Delayed Processing

```python
import boto3
import json
from datetime import datetime, timedelta

scheduler = boto3.client('scheduler', region_name='us-east-1')

def schedule_payment_reminder(order_id: str, reminder_time: datetime):
    """
    Schedule a one-time event to fire at a specific time.
    Useful for payment reminders, SLA checks, delayed notifications.
    """
    scheduler.create_schedule(
        Name=f"payment-reminder-{order_id}",
        ScheduleExpression=f"at({reminder_time.strftime('%Y-%m-%dT%H:%M:%S')})",
        FlexibleTimeWindow={'Mode': 'OFF'},
        Target={
            'Arn': 'arn:aws:lambda:us-east-1:123456789012:function:payment-reminder',
            'RoleArn': 'arn:aws:iam::123456789012:role/scheduler-invoke-role',
            'Input': json.dumps({
                'orderId': order_id,
                'action': 'send_payment_reminder'
            })
        },
        ActionAfterCompletion='DELETE'  # Clean up after firing
    )
```

### Lambda Consumer (SQS Event Source)

```python
import json

def lambda_handler(event, context):
    """
    Lambda function triggered by SQS event source mapping.
    AWS handles polling, batching, and scaling consumer concurrency.
    This is the recommended way to consume SQS in serverless architectures.
    """
    batch_item_failures = []

    for record in event['Records']:
        try:
            # If the SQS queue is subscribed to SNS, unwrap the envelope
            body = json.loads(record['body'])
            if 'Message' in body and 'TopicArn' in body:
                # SNS envelope
                message = json.loads(body['Message'])
            else:
                # Direct SQS message
                message = body

            process_event(message)

        except Exception as e:
            # Report individual item failure for partial batch response
            # Requires ReportBatchItemFailures in the event source mapping
            batch_item_failures.append({
                'itemIdentifier': record['messageId']
            })

    return {
        'batchItemFailures': batch_item_failures
    }


def process_event(event: dict):
    """Your business logic. Raise an exception to signal failure."""
    print(f"Processing: {json.dumps(event)}")
```

---

## Cost Analysis at Various Scales

The following estimates assume us-east-1 pricing as of 2025 and use the SNS+SQS fan-out pattern with three consumers.

| Scale | Events/month | SNS cost | SQS cost (3 queues) | EventBridge cost | Estimated total |
|-------|-------------|----------|---------------------|-----------------|-----------------|
| Startup | 1M | $0.50 | ~$2.40 | $1.00 | ~$4/mo |
| Growth | 100M | $50 | ~$240 | $100 | ~$390/mo |
| Scale | 1B | $500 | ~$2,400 | $1,000 | ~$3,900/mo |
| Enterprise | 10B | $5,000 | ~$24,000 | $10,000 | ~$39,000/mo |

At startup scale, AWS managed services are essentially free. At enterprise scale, you are paying $39,000/month for what a self-hosted Kafka cluster might cost $15,000/month to run (including engineer time, which is the number people always conveniently forget when comparing). The breakeven point depends entirely on how you value your team's time spent *not* operating infrastructure.

The SQS cost is the largest line item because of the per-request pricing model. Each message consumed involves at minimum a ReceiveMessage and a DeleteMessage call. With three consumer queues, each event generates at least six SQS API calls plus the original SNS publish. At 10 billion events per month, that is 60+ billion SQS API calls.

**Hidden costs to watch:**
- CloudWatch Logs from Lambda consumers (can dwarf the messaging costs)
- KMS charges if using customer-managed keys ($1/month per key + $0.03 per 10,000 requests)
- Data transfer if sending events cross-region ($0.01–0.02 per GB)

---

## Integration with Lambda, Step Functions, and ECS

**Lambda** is the natural consumer for all three services. SQS event source mappings handle polling, batching, and scaling automatically. SNS can invoke Lambda directly (though routing through SQS first gives you better error handling and DLQ support). EventBridge can target Lambda as a rule target. The Lambda service manages consumer concurrency — it will scale up invocations as queue depth increases, up to your concurrency limit.

**Step Functions** integrate with EventBridge through the `.waitForTaskToken` pattern, allowing long-running workflows to pause and resume based on events. Step Functions can also publish events to EventBridge as a workflow step. This combination is powerful for orchestrating multi-step processes that span multiple services and require human approval, external callbacks, or time-based waits.

**ECS** (and EKS) consumers poll SQS directly using the SDK, typically running as long-lived services with multiple threads or processes. This is the right model when your consumer requires persistent connections to downstream systems, heavy initialisation, or more control over concurrency than Lambda provides. ECS auto-scaling based on SQS queue depth (using `ApproximateNumberOfMessagesVisible` as a CloudWatch metric for target tracking scaling) works well, though the scaling response time is measured in minutes, not seconds.

---

## Verdict

SNS, SQS, and EventBridge are the sensible default for event-driven architecture on AWS. Not the fastest. Not the most flexible. Not the cheapest at massive scale. But the most *practical* for the overwhelming majority of workloads.

The zero-ops model is real and valuable. The pay-per-use pricing makes experimentation cheap. The integration with the rest of the AWS ecosystem is unmatched. And the durability and availability of these services — battle-tested over nearly two decades in SQS's case — is not something you replicate easily with self-hosted alternatives.

The trade-offs are equally real. You are locked into AWS. Your throughput ceilings for ordered messaging are low. Your replay capabilities are limited. Your end-to-end latency will be measured in tens or hundreds of milliseconds, not single digits. And at truly massive scale, the per-request pricing model makes the cost conversation interesting.

The honest recommendation: if you are building on AWS (and you have already made that decision), start with SNS+SQS for fan-out and EventBridge for routing. You will get a functioning, production-grade event-driven architecture in days rather than weeks, and your on-call engineers will thank you for not giving them a Kafka cluster to babysit. If and when you hit the limits — and you will know when you hit them — you can evaluate Kafka, Kinesis, or MSK for the specific workloads that need more muscle.

Do not over-engineer your messaging layer. The most reliable message broker is the one you do not have to operate.
