# Event-Driven Architecture

A technically rigorous guide to event-driven architecture and the message broker landscape — with dry wit and honest assessments instead of vendor brochures.

**[Read the book →](https://cloudstreet-dev.github.io/Event-Driven-Architecture/)**

## What's Inside

### Part 1: Event-Driven Architecture Deep Dive

Covers the fundamentals without pulling punches: core concepts, patterns (pub/sub, event sourcing, CQRS, sagas), schema evolution, error handling and delivery guarantees (including why "exactly-once" is a polite fiction), observability, security, testing, and a field guide to anti-patterns.

### Part 2: The Broker Showdown

An opinionated evaluation of 16+ message brokers using a consistent framework — architecture, strengths, weaknesses, operational reality, code examples, and a verdict for each:

| Broker | One-Liner |
|--------|-----------|
| **Apache Kafka** | The 800-pound gorilla |
| **RabbitMQ** | The reliable workhorse |
| **Apache Pulsar** | The multi-tenant challenger |
| **AWS SNS/SQS & EventBridge** | The AWS-native path |
| **Google Pub/Sub & Azure Event Hubs** | The other cloud-native options |
| **Redis Streams** | When your cache gets ambitious |
| **NATS & JetStream** | The lightweight speed demon |
| **ActiveMQ & Artemis** | The enterprise veterans |
| **ZeroMQ** | The brokerless broker |
| **Redpanda** | Kafka without the JVM tax |
| **Memphis** | The developer-experience play |
| **Solace PubSub+** | The enterprise dark horse |
| **Chronicle Queue** | When microseconds matter |
| **Aeron** | For the ultra-low-latency crowd |

Plus a chapter on the obscure and curious (QStash, Watermill, EventStoreDB, RocketMQ, and more), a full comparison matrix, and a selection guide with decision trees.

## Building Locally

Requires [mdBook](https://rust-lang.github.io/mdBook/guide/installation.html):

```sh
mdbook serve --open
```

## License

See [LICENSE](LICENSE).
