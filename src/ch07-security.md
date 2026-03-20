# Security and Access Control

You built a beautiful event-driven system. Microservices hum along, events flow like water, and your architecture diagram looks like something out of a conference talk. Then someone points out that every service can read every topic, your PII is sitting in plaintext in an append-only log that you promised was immutable, and your GDPR compliance officer has started drinking at lunch.

Welcome to security in event-driven systems — where the attack surface is larger, the blast radius is wider, and the consequences of getting it wrong are distributed across every consumer that ever touched the data.

---

## The Expanded Attack Surface

In a traditional request-response system, your security perimeter is relatively well-defined. You have API gateways, authentication middleware, and a clear sense of who is talking to whom. In an event-driven system, you have... more.

Consider what you're actually defending:

- **The broker itself** — a centralized nervous system that, if compromised, gives an attacker access to every conversation in your organization.
- **Producers** — any service that can write to a topic can inject poisoned events that every downstream consumer will dutifully process.
- **Consumers** — any service that can read from a topic gets access to every event ever published there, potentially including historical data going back years.
- **The network between all of the above** — events in transit are just bytes on a wire, and bytes on a wire can be read.
- **The storage layer** — events at rest on broker disks, in consumer state stores, in dead letter queues, in replay buffers. Your data has more copies than a bestselling novel.
- **The schema registry** — whoever controls the schema controls what producers can say and what consumers expect. Schema poisoning is a real attack vector.

The fundamental problem is this: event-driven architectures trade direct service-to-service communication for indirect communication through a shared medium. That shared medium becomes a high-value target. It's the difference between intercepting a phone call between two people and tapping the entire telephone exchange.

### Threat Modeling for EDA

If you're not doing threat modeling for your event-driven systems, you're not doing security — you're doing hope. The STRIDE framework adapts well:

| Threat | EDA Manifestation |
|--------|-------------------|
| **Spoofing** | A rogue producer impersonates a legitimate service and publishes fraudulent events |
| **Tampering** | Events are modified in transit or an attacker alters committed events on disk |
| **Repudiation** | A producer denies publishing an event; no audit trail to prove otherwise |
| **Information Disclosure** | A consumer reads topics it shouldn't; PII leaks through overly broad subscriptions |
| **Denial of Service** | A producer floods a topic, overwhelming consumers; a consumer creates excessive lag |
| **Elevation of Privilege** | A service with read-only access gains write access to a topic; a consumer modifies broker configuration |

---

## Authentication and Authorization for Producers and Consumers

### Authentication: Proving You Are Who You Claim

Every client connecting to your broker — producer or consumer — needs to prove its identity. The days of "it's on the internal network, so it's fine" ended roughly around the time that the concept of "internal network" became a polite fiction.

**SASL (Simple Authentication and Security Layer)** is the most common framework for broker authentication. The name contains the word "Simple," which should immediately make you suspicious. It supports multiple mechanisms:

- **SASL/PLAIN** — username and password sent in cleartext. Only acceptable over TLS. If you're using this without TLS, you don't have authentication; you have a suggestion.
- **SASL/SCRAM** (Salted Challenge Response Authentication Mechanism) — challenge-response protocol that avoids sending the password over the wire. SHA-256 or SHA-512. A meaningful improvement over PLAIN.
- **SASL/GSSAPI (Kerberos)** — enterprise-grade authentication via Kerberos tickets. If your organization already runs Active Directory, this integrates naturally. If it doesn't, setting up Kerberos just for your message broker is a special kind of masochism.
- **SASL/OAUTHBEARER** — OAuth 2.0 bearer tokens. The modern choice for organizations that have already invested in an identity provider. Tokens are short-lived and can carry fine-grained claims.

**mTLS (Mutual TLS)** — both client and server present certificates. This is the gold standard for service-to-service authentication in event-driven systems. Each service gets its own certificate, and the broker verifies it before allowing any operations.

```yaml
# Kafka broker configuration for mTLS
listeners=SSL://broker1:9093
ssl.keystore.location=/var/kafka/ssl/kafka.server.keystore.jks
ssl.keystore.password=${KEYSTORE_PASSWORD}
ssl.key.password=${KEY_PASSWORD}
ssl.truststore.location=/var/kafka/ssl/kafka.server.truststore.jks
ssl.truststore.password=${TRUSTSTORE_PASSWORD}
ssl.client.auth=required
ssl.endpoint.identification.algorithm=https

# Map the Distinguished Name from client certs to a Kafka principal
ssl.principal.mapping.rules=RULE:^CN=([a-zA-Z0-9._-]+),.*$/$$1/,DEFAULT
```

```yaml
# Kafka producer configuration for mTLS
bootstrap.servers=broker1:9093
security.protocol=SSL
ssl.keystore.location=/var/app/ssl/producer.keystore.jks
ssl.keystore.password=${KEYSTORE_PASSWORD}
ssl.key.password=${KEY_PASSWORD}
ssl.truststore.location=/var/app/ssl/ca.truststore.jks
ssl.truststore.password=${TRUSTSTORE_PASSWORD}
ssl.endpoint.identification.algorithm=https
```

```python
# Python producer with mTLS
from confluent_kafka import Producer

producer = Producer({
    'bootstrap.servers': 'broker1:9093',
    'security.protocol': 'SSL',
    'ssl.ca.location': '/var/app/ssl/ca-cert.pem',
    'ssl.certificate.location': '/var/app/ssl/client-cert.pem',
    'ssl.key.location': '/var/app/ssl/client-key.pem',
    'ssl.key.password': os.environ['SSL_KEY_PASSWORD'],
    'ssl.endpoint.identification.algorithm': 'https',
})
```

> **Certificate Management Reality Check:** mTLS is excellent security and terrible operations. You now need to provision, distribute, rotate, and revoke certificates for every service. You need a PKI (Public Key Infrastructure), or at minimum a tool like HashiCorp Vault, cert-manager (in Kubernetes), or a service mesh that handles this for you. If your plan is "we'll manage the certs manually," your plan is to have an outage on the day a cert expires and nobody noticed.

### Authorization: Proving You're Allowed to Do What You're Trying to Do

Authentication tells you *who*. Authorization tells you *what they can do*. These are frequently conflated by people who should know better.

---

## Topic-Level and Event-Level Access Control

### ACLs (Access Control Lists)

The most straightforward model. Each topic has a list of principals (users or services) and the operations they're allowed to perform.

```bash
# Kafka ACL examples

# Allow the order-service to produce to the orders topic
kafka-acls --bootstrap-server broker1:9093 \
  --command-config admin.properties \
  --add \
  --allow-principal User:order-service \
  --operation Write \
  --topic orders

# Allow the shipping-service to consume from the orders topic
kafka-acls --bootstrap-server broker1:9093 \
  --command-config admin.properties \
  --add \
  --allow-principal User:shipping-service \
  --operation Read \
  --topic orders \
  --group shipping-consumer-group

# Deny all other access to the orders topic (deny by default)
kafka-acls --bootstrap-server broker1:9093 \
  --command-config admin.properties \
  --add \
  --deny-principal User:* \
  --operation All \
  --topic orders
```

ACLs work. They're simple to understand, simple to audit, and simple to get wrong at scale. When you have 50 services and 200 topics, you have potentially 10,000 ACL entries to manage. This is when people start looking at RBAC.

### RBAC (Role-Based Access Control)

Instead of granting permissions to individual services, you define roles and assign services to roles.

```
# Role definitions (conceptual)
role: order-writer
  permissions:
    - topic: orders
      operations: [Write, Describe]
    - topic: order-events
      operations: [Write, Describe]

role: order-reader
  permissions:
    - topic: orders
      operations: [Read, Describe]
    - topic: order-events
      operations: [Read, Describe]

# Role assignments
principal: order-service     -> roles: [order-writer]
principal: shipping-service  -> roles: [order-reader]
principal: billing-service   -> roles: [order-reader]
principal: analytics-service -> roles: [order-reader]
```

Confluent Platform offers built-in RBAC. Open-source Kafka does not — you'll need to build or buy an authorization plugin that implements the `Authorizer` interface. RabbitMQ has a plugin-based authorization model. Pulsar has a multi-tenant authorization model built in. The managed cloud brokers (AWS MSK, Confluent Cloud, etc.) generally have RBAC as a feature.

### ABAC (Attribute-Based Access Control)

The most flexible and most complex model. Access decisions are based on attributes of the principal, the resource, the action, and the environment.

```
# ABAC policy (conceptual)
PERMIT if:
  subject.department == "finance" AND
  resource.topic.classification == "financial" AND
  action == "Read" AND
  environment.time.hour BETWEEN 6 AND 22 AND
  environment.network.zone == "corporate"
```

ABAC is powerful. It can express policies that RBAC cannot, such as time-based restrictions or data classification rules. It's also significantly harder to reason about, audit, and debug. When an engineer at 2 AM is trying to figure out why a consumer can't read from a topic, "check the attribute-based policy engine" is not the answer they want to hear.

**Practical guidance:** Start with ACLs. Move to RBAC when ACL management becomes painful. Move to ABAC only when you have a genuine requirement that RBAC cannot express and you have the tooling and expertise to manage it. Most organizations never need ABAC for their event infrastructure.

### Event-Level Access Control

Topic-level access control is coarse-grained. What if different events on the same topic have different sensitivity levels? An `OrderPlaced` event might be fine for the analytics team, but an `OrderPaymentProcessed` event contains card details they shouldn't see.

Options:

1. **Separate topics by sensitivity** — the simplest approach. Put sensitive events on a restricted topic. This works but leads to topic proliferation.
2. **Field-level encryption** — encrypt sensitive fields within events. Consumers without the decryption key see ciphertext. More on this below.
3. **Event-level authorization in the consumer** — the consumer checks whether it's authorized to process each event type. This is enforcement at the wrong layer and depends on consumers being honest, which is the security equivalent of the honor system.
4. **Broker-side filtering with authorization** — some brokers support server-side filtering (Pulsar's message filtering, for example). You can combine this with authorization to prevent certain events from being delivered to certain consumers. This is broker-specific and often limited.

---

## Encryption: In-Transit and At-Rest

### Encryption In-Transit (TLS/mTLS)

If you're running a production message broker without TLS, stop reading this chapter and go fix that. I'll wait.

TLS encrypts the communication channel between producers, consumers, and the broker. mTLS adds mutual authentication (covered above). The configuration is straightforward but the operational overhead is real.

```properties
# Kafka broker TLS configuration
listeners=SSL://0.0.0.0:9093
advertised.listeners=SSL://broker1.example.com:9093

ssl.keystore.type=PKCS12
ssl.keystore.location=/etc/kafka/ssl/broker.keystore.p12
ssl.keystore.password=${KEYSTORE_PASSWORD}
ssl.key.password=${KEY_PASSWORD}

ssl.truststore.type=PKCS12
ssl.truststore.location=/etc/kafka/ssl/truststore.p12
ssl.truststore.password=${TRUSTSTORE_PASSWORD}

# TLS version — TLSv1.3 if your JVM supports it
ssl.enabled.protocols=TLSv1.3,TLSv1.2
ssl.protocol=TLSv1.3

# Cipher suites — be explicit, don't rely on defaults
ssl.cipher.suites=TLS_AES_256_GCM_SHA384,TLS_CHACHA20_POLY1305_SHA256
```

**Performance impact:** TLS adds CPU overhead for encryption/decryption. On modern hardware with AES-NI instructions, the impact is typically 5-15% throughput reduction. If this is unacceptable, you have unusual requirements or unusual hardware. The answer is never "skip TLS." The answer is "get better hardware" or "use TLS offloading."

> **Inter-broker communication:** Don't forget to encrypt communication between broker nodes. In a Kafka cluster, replication traffic between brokers can contain the same sensitive data as producer/consumer traffic. Configure `inter.broker.listener.name` to use an SSL listener.

### Encryption At-Rest

TLS protects data in motion. It does nothing for data sitting on the broker's disks. If someone gains access to the broker's filesystem — through a compromised host, a stolen backup, or a decommissioned disk that wasn't properly wiped — they can read every event in plaintext.

**Broker-level disk encryption:**

- **Full-disk encryption** (LUKS, BitLocker, AWS EBS encryption) — transparent to the broker, protects against physical disk theft and improper decommissioning. Does not protect against a compromised broker process or an attacker with OS-level access.
- **Filesystem-level encryption** (eCryptfs, fscrypt) — similar protection to full-disk, slightly more targeted.

**Broker-native encryption at rest:**

- Confluent Platform offers transparent encryption at rest.
- AWS MSK offers encryption at rest via KMS.
- Most managed broker services offer this as a checkbox. Check the checkbox.

**Application-level encryption (end-to-end):**

The producer encrypts the event payload before publishing; the consumer decrypts after consuming. The broker never sees plaintext. This is the strongest model but requires key management at the application layer.

```java
// Application-level envelope encryption for Kafka events
public class EncryptingSerializer implements Serializer<Event> {
    private final KmsClient kmsClient;
    private final String masterKeyId;

    @Override
    public byte[] serialize(String topic, Event event) {
        // 1. Generate a data encryption key (DEK)
        GenerateDataKeyResponse dataKey = kmsClient.generateDataKey(
            GenerateDataKeyRequest.builder()
                .keyId(masterKeyId)
                .keySpec(DataKeySpec.AES_256)
                .build()
        );

        // 2. Encrypt the event payload with the DEK
        byte[] plaintext = jsonSerializer.serialize(event);
        byte[] ciphertext = aesEncrypt(plaintext, dataKey.plaintext().asByteArray());

        // 3. Package the encrypted DEK alongside the ciphertext
        EncryptedEnvelope envelope = new EncryptedEnvelope(
            dataKey.ciphertextBlob().asByteArray(),  // encrypted DEK
            ciphertext,                                // encrypted payload
            "AES-256-GCM",                            // algorithm
            masterKeyId                                // key reference
        );

        return envelopeSerializer.serialize(envelope);
    }
}
```

The downside: the broker cannot inspect, filter, route, or compact based on encrypted payloads. You lose broker-side processing capabilities. Compaction, in particular, becomes problematic — the broker can't determine which events share a key if the key is encrypted.

---

## Field-Level Encryption for Sensitive Data in Events

Full-payload encryption is a blunt instrument. Most of your event data isn't sensitive — it's the two or three fields containing email addresses, phone numbers, or payment details that need protection. Field-level encryption lets you encrypt only the sensitive fields, leaving the rest in plaintext for routing, filtering, and debugging.

```json
{
  "eventType": "OrderPlaced",
  "orderId": "ord-12345",
  "timestamp": "2025-11-15T10:30:00Z",
  "customerId": "cust-789",
  "customerEmail": "ENC[AES256-GCM:AwEBAQx2...base64...]",
  "customerPhone": "ENC[AES256-GCM:BxFCAgR3...base64...]",
  "shippingAddress": "ENC[AES256-GCM:CyGDCgS4...base64...]",
  "items": [
    {"sku": "WIDGET-001", "quantity": 2, "price": 29.99}
  ],
  "totalAmount": 59.98,
  "currency": "USD"
}
```

The event is still routable by `orderId`, filterable by `eventType`, and inspectable for debugging — but the PII fields are opaque to anyone without the decryption key.

### Implementation Approaches

**Approach 1: Custom serializer/deserializer**

```python
from cryptography.fernet import Fernet
import json
import os

class FieldLevelEncryption:
    """Encrypts specified fields in an event payload."""

    def __init__(self, key: bytes, sensitive_fields: list[str]):
        self.fernet = Fernet(key)
        self.sensitive_fields = set(sensitive_fields)

    def encrypt_event(self, event: dict) -> dict:
        encrypted = event.copy()
        for field in self.sensitive_fields:
            if field in encrypted and encrypted[field] is not None:
                plaintext = str(encrypted[field]).encode('utf-8')
                encrypted[field] = f"ENC[{self.fernet.encrypt(plaintext).decode('utf-8')}]"
        return encrypted

    def decrypt_event(self, event: dict) -> dict:
        decrypted = event.copy()
        for field in self.sensitive_fields:
            if field in decrypted and isinstance(decrypted[field], str) \
               and decrypted[field].startswith("ENC["):
                token = decrypted[field][4:-1].encode('utf-8')
                decrypted[field] = self.fernet.decrypt(token).decode('utf-8')
        return decrypted


# Usage
encryptor = FieldLevelEncryption(
    key=os.environ['FIELD_ENCRYPTION_KEY'].encode(),
    sensitive_fields=['customerEmail', 'customerPhone', 'shippingAddress']
)

# Producer side
raw_event = {
    "eventType": "OrderPlaced",
    "orderId": "ord-12345",
    "customerEmail": "alice@example.com",
    "customerPhone": "+1-555-0123",
    "shippingAddress": "123 Main St, Springfield",
    "totalAmount": 59.98
}
encrypted_event = encryptor.encrypt_event(raw_event)
producer.produce(topic='orders', value=json.dumps(encrypted_event))

# Consumer side (authorized consumer with the key)
decrypted_event = encryptor.decrypt_event(encrypted_event)
# decrypted_event['customerEmail'] == 'alice@example.com'

# Consumer side (unauthorized consumer without the key)
# They see: "ENC[gAAAAABh...]" and can do nothing with it
```

**Approach 2: Schema-driven encryption**

Define which fields are sensitive in the schema itself, and let the serialization layer handle encryption automatically.

```json
{
  "type": "record",
  "name": "OrderPlaced",
  "namespace": "com.example.orders",
  "fields": [
    {"name": "orderId", "type": "string"},
    {"name": "customerEmail", "type": "string",
     "confluent:tags": ["PII", "ENCRYPTED"]},
    {"name": "customerPhone", "type": "string",
     "confluent:tags": ["PII", "ENCRYPTED"]},
    {"name": "totalAmount", "type": "double"}
  ]
}
```

Confluent's schema-level encryption and similar tools use the schema metadata to determine which fields to encrypt, using keys managed through a KMS. This is cleaner than hand-rolling encryption in every producer and consumer, but it ties you to a specific vendor's ecosystem.

### Key Management for Field-Level Encryption

The encryption is only as good as the key management. Options:

- **Envelope encryption via KMS (AWS KMS, GCP KMS, Azure Key Vault, HashiCorp Vault):** A master key in the KMS encrypts per-field data encryption keys. Producers request a DEK from the KMS, encrypt the field, and store the encrypted DEK alongside the data. Consumers retrieve the encrypted DEK, call the KMS to decrypt it, and use it to decrypt the field. This is the correct approach.
- **Shared symmetric key in environment variables:** Works for small deployments. Rotation is painful. If the key leaks, every event ever encrypted with it is compromised. Not recommended for production.
- **Per-consumer keys:** Different consumers get different keys, enabling differential access. The producer encrypts sensitive fields multiple times (once per authorized consumer's key) or uses an intermediary re-encryption service. Complex but powerful.

---

## PII in Event Payloads — GDPR's Revenge on Event Sourcing

Here is where event-driven architecture and data protection regulation have a philosophical disagreement.

Event-driven systems love immutability. Events are facts. They happened. You don't change them. This is a core architectural principle, and it gives you audit trails, replay capability, and temporal queries.

GDPR (and CCPA, LGPD, PIPEDA, and the growing family of privacy regulations) loves the right to erasure. Individuals can request that their personal data be deleted. All of it. Including the copy you made three years ago in that event log you forgot about.

These two principles are, on their face, incompatible. And yet, here you are, needing to satisfy both.

### The Scope of the Problem

PII leaks into event payloads in ways you don't expect:

- **Obvious:** `customerEmail`, `customerName`, `shippingAddress`
- **Less obvious:** `ipAddress` in click events, `userAgent` strings, GPS coordinates in location events
- **Sneaky:** Free-text fields like `orderNotes` ("Please deliver to Alice Smith at 123 Main St"), correlation IDs that embed user IDs, URLs that contain email addresses as query parameters

If you're doing event sourcing, the problem is worse. Your entire state is derived from events. Deleting an event doesn't just remove data — it potentially corrupts the state of every downstream projection.

### Strategies for PII Compliance

**Strategy 1: Don't store PII in events.**

The simplest approach: events reference external entities by opaque ID rather than embedding PII.

```json
// BAD: PII embedded in event
{
  "eventType": "OrderPlaced",
  "customerName": "Alice Smith",
  "customerEmail": "alice@example.com",
  "shippingAddress": "123 Main St, Springfield, IL 62701"
}

// BETTER: PII referenced by ID
{
  "eventType": "OrderPlaced",
  "customerId": "cust-a1b2c3",
  "shippingAddressId": "addr-x9y8z7"
}
```

The PII lives in a mutable data store (a database) where it can be updated or deleted. Events contain only references. When a consumer needs the PII, it looks it up — and if the data has been deleted, the lookup returns nothing.

The downside: you lose the self-contained nature of events. Consumers now need access to external data stores. Replay becomes complicated because the external data may have changed since the event was produced. You've traded one problem for another, but the new problem is at least one that databases have been solving for decades.

**Strategy 2: Crypto-shredding (the practical answer).**

This deserves its own section. See below.

**Strategy 3: Event transformation and redaction pipelines.**

A dedicated service sits between the raw event stream and downstream consumers, stripping or masking PII fields before forwarding events. The raw stream is access-controlled and retained only as long as legally required; the redacted stream is what most consumers see.

```
Producer -> [raw-orders topic] -> PII Redaction Service -> [clean-orders topic] -> Consumers
                                        |
                                  (strips PII,
                                   replaces with
                                   hashes or tokens)
```

This works but introduces an additional service to maintain, a latency overhead, and a risk that the redaction logic misses a field.

---

## The Right to Be Forgotten vs. Immutable Event Logs — The Crypto-Shredding Pattern

Crypto-shredding is the industry's best answer to the immutability-vs-deletion paradox. The idea is elegant:

1. All PII in events is encrypted with a key that is unique to the data subject (the person whose data it is).
2. When that person exercises their right to erasure, you don't delete the events — you delete the encryption key.
3. Without the key, the encrypted PII fields are indistinguishable from random bytes. The data is effectively destroyed while the event structure remains intact.

### Implementation

```python
class CryptoShredding:
    """
    Per-subject encryption keys for PII in events.
    Deleting the key == deleting the data.
    """

    def __init__(self, key_store):
        """
        key_store: a durable, secure store mapping subject_id -> encryption_key.
        Could be HashiCorp Vault, AWS KMS, a dedicated database, etc.
        """
        self.key_store = key_store

    def get_or_create_key(self, subject_id: str) -> bytes:
        """Get the encryption key for a data subject, creating one if needed."""
        key = self.key_store.get(subject_id)
        if key is None:
            key = Fernet.generate_key()
            self.key_store.put(subject_id, key)
        return key

    def encrypt_pii(self, subject_id: str, event: dict,
                    pii_fields: list[str]) -> dict:
        """Encrypt PII fields using the subject's key."""
        key = self.get_or_create_key(subject_id)
        fernet = Fernet(key)

        encrypted = event.copy()
        encrypted['_pii_subject'] = subject_id  # track whose key to use
        encrypted['_pii_fields'] = pii_fields   # track which fields are encrypted

        for field in pii_fields:
            if field in encrypted:
                plaintext = json.dumps(encrypted[field]).encode('utf-8')
                encrypted[field] = fernet.encrypt(plaintext).decode('utf-8')

        return encrypted

    def decrypt_pii(self, event: dict) -> dict:
        """Decrypt PII fields. Returns event with '[DELETED]' if key is gone."""
        subject_id = event.get('_pii_subject')
        pii_fields = event.get('_pii_fields', [])

        if not subject_id:
            return event

        key = self.key_store.get(subject_id)

        decrypted = event.copy()
        for field in pii_fields:
            if field in decrypted:
                if key is None:
                    # Key has been shredded — data is effectively deleted
                    decrypted[field] = '[DELETED]'
                else:
                    fernet = Fernet(key)
                    plaintext = fernet.decrypt(decrypted[field].encode('utf-8'))
                    decrypted[field] = json.loads(plaintext.decode('utf-8'))

        return decrypted

    def forget_subject(self, subject_id: str):
        """
        Exercise the right to be forgotten.
        Delete the key, and all PII for this subject becomes unrecoverable.
        """
        self.key_store.delete(subject_id)
        # That's it. Every event containing this subject's PII
        # is now cryptographically shredded.
```

### Crypto-Shredding Considerations

- **Key storage is critical.** The key store is now the most important database in your system. If you lose the keys accidentally, you've accidentally GDPR-deleted all your customer data. Back it up. Replicate it. Treat it like the crown jewels it is.
- **Key rotation.** If you rotate a subject's key, you need to re-encrypt all events containing that subject's PII with the new key. For event sourcing, this means rewriting history — which you said you wouldn't do, but here we are.
- **Downstream copies.** Crypto-shredding works for the event log. It doesn't help with consumers that decrypted the PII and stored it in their own databases. You need a coordinated deletion process across all consumers. This is the part that makes compliance officers nervous.
- **Performance.** Per-subject keys mean a KMS lookup for every event containing PII. Caching helps but introduces a window where a deleted key might still be cached. Set reasonable TTLs.
- **Legal acceptance.** Check with your legal team whether your regulators consider crypto-shredding equivalent to deletion. Most European DPAs accept it, but "most" is not "all."

---

## Audit Trails and Compliance

Event-driven systems have a natural advantage for audit trails: they already record what happened. The challenge is making that record trustworthy, complete, and queryable.

### What to Audit

- **Producer actions:** Who published what, when, to which topic. Include the producer's authenticated identity, the event type, a timestamp, and enough metadata to reconstruct the action.
- **Consumer actions:** Who consumed what, when. This is harder to capture since consumption is typically a pull operation, but broker-side access logs can provide it.
- **Administrative actions:** Topic creation/deletion, ACL changes, schema updates, configuration changes. These are the actions an attacker would take to cover their tracks.
- **Access denials:** Failed authentication attempts, authorization failures, rate limit hits. These are often more interesting than successful operations.

### Implementing Audit Trails

```java
// Interceptor-based audit logging for Kafka producers
public class AuditProducerInterceptor implements ProducerInterceptor<String, byte[]> {

    private final AuditLogger auditLogger;

    @Override
    public ProducerRecord<String, byte[]> onSend(ProducerRecord<String, byte[]> record) {
        auditLogger.logProduceAttempt(
            AuditEvent.builder()
                .principal(SecurityContext.getCurrentPrincipal())
                .action("PRODUCE")
                .topic(record.topic())
                .partition(record.partition())
                .key(record.key())
                .eventType(extractEventType(record.headers()))
                .timestamp(Instant.now())
                .sourceIp(SecurityContext.getSourceIp())
                .payloadSizeBytes(record.value().length)
                // Do NOT log the payload itself — that defeats the purpose
                // of field-level encryption
                .build()
        );
        return record;
    }

    @Override
    public void onAcknowledgement(RecordMetadata metadata, Exception exception) {
        if (exception != null) {
            auditLogger.logProduceFailure(metadata, exception);
        } else {
            auditLogger.logProduceSuccess(metadata);
        }
    }
}
```

**Do not log event payloads in audit trails.** This seems counterintuitive, but the audit trail's purpose is to record *who did what*, not to create yet another copy of the data. Logging payloads creates a second unencrypted copy of PII outside your carefully encrypted event stream. Log metadata only: event type, topic, key, timestamp, principal, outcome.

### Tamper-Evident Audit Logs

An attacker who compromises your system will attempt to modify audit logs to cover their tracks. Options for tamper evidence:

- **Append-only storage** — write audit logs to a system that doesn't support mutation (S3 with Object Lock, WORM storage).
- **Hash chaining** — each audit entry includes a hash of the previous entry, creating a blockchain-like chain. Tampering with any entry invalidates all subsequent hashes.
- **External attestation** — periodically send a hash of your audit log to an external timestamping service. This proves the log existed in a particular state at a particular time.

---

## Schema Validation as a Security Boundary

Your schema registry isn't just a convenience for managing data formats. It's a security boundary. A schema defines what a valid event looks like, and rejecting events that don't conform to the schema is a form of input validation — the most fundamental security control there is.

### Schema Validation as Input Filtering

Without schema validation, a malicious producer can:

- Inject oversized events that exhaust consumer memory
- Include unexpected fields that trigger deserialization vulnerabilities
- Embed malicious content (script injection payloads, SQL injection strings) in text fields
- Send malformed data that causes consumer crashes (null pointer exceptions, type confusion)

With schema validation enforced at the broker or serialization layer:

```json
{
  "type": "record",
  "name": "OrderPlaced",
  "namespace": "com.example.orders",
  "fields": [
    {
      "name": "orderId",
      "type": "string",
      "doc": "UUID format order identifier",
      "pattern": "^ord-[a-f0-9]{8}$"
    },
    {
      "name": "totalAmount",
      "type": "double",
      "min": 0.0,
      "max": 1000000.0
    },
    {
      "name": "currency",
      "type": {
        "type": "enum",
        "name": "Currency",
        "symbols": ["USD", "EUR", "GBP"]
      }
    }
  ]
}
```

Schema validation won't stop a determined attacker, but it raises the bar significantly. It's the event-driven equivalent of parameterized queries — it doesn't solve every security problem, but not using it is inexcusable.

---

## Securing Schema Registries

The schema registry is a control plane component. Whoever controls the schema controls the contract between producers and consumers. An attacker who can modify a schema can:

- Add fields that legitimate consumers don't expect, potentially causing crashes.
- Change field types to trigger deserialization vulnerabilities.
- Remove required fields, breaking downstream processing.
- Weaken validation constraints, allowing malicious payloads through.

### Hardening the Schema Registry

1. **Authentication and authorization:** The schema registry should require authentication. Not all users need the same access. Producers need read access (to validate against the current schema). Schema administrators need write access. Consumers need read access. Nobody else needs any access.

2. **Change control:** Schema changes should go through a review process, not be applied directly by producers at runtime. Treat schema changes like database migrations — reviewed, tested, and deployed through a pipeline.

3. **Compatibility enforcement:** Enable strict compatibility checking (backward, forward, or full compatibility). This prevents breaking changes from being registered even if an attacker gains write access.

4. **Network isolation:** The schema registry should not be accessible from the public internet. It should be on a private network, accessible only to services that need it.

5. **Audit logging:** Log every schema read and write operation. Alert on unexpected schema modifications.

```bash
# Confluent Schema Registry — enable authentication
# In schema-registry.properties:
authentication.method=BASIC
authentication.roles=admin,developer,readonly
authentication.realm=SchemaRegistry

# Enable HTTPS
listeners=https://0.0.0.0:8081
ssl.keystore.location=/etc/schema-registry/ssl/keystore.p12
ssl.keystore.password=${KEYSTORE_PASSWORD}
ssl.truststore.location=/etc/schema-registry/ssl/truststore.p12
ssl.truststore.password=${TRUSTSTORE_PASSWORD}
```

---

## Network Segmentation and Broker Hardening

### Network Architecture

The broker cluster should live in a dedicated network segment, separated from application services by firewalls or security groups. The principle of least privilege applies at the network level:

```
┌─────────────────────────────────────────────────┐
│                   Internet                       │
└──────────────────────┬──────────────────────────┘
                       │ (no direct access)
┌──────────────────────┴──────────────────────────┐
│              API Gateway / Load Balancer          │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────┴──────────────────────────┐
│              Application Services Network         │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │ Service A  │  │ Service B  │  │ Service C  │    │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘    │
└────────┼──────────────┼──────────────┼──────────┘
         │              │              │
    ┌────┴──────────────┴──────────────┴────┐
    │     Broker Network (restricted)        │
    │  ┌─────────┐ ┌─────────┐ ┌─────────┐  │
    │  │Broker 1 │ │Broker 2 │ │Broker 3 │  │
    │  └─────────┘ └─────────┘ └─────────┘  │
    │  ┌──────────────────────────────────┐  │
    │  │      ZooKeeper / KRaft           │  │
    │  └──────────────────────────────────┘  │
    └────────────────────────────────────────┘
```

### Broker Hardening Checklist

- [ ] Disable all unauthenticated listeners. No PLAINTEXT listeners in production. Zero.
- [ ] Enable TLS for all client connections and inter-broker communication.
- [ ] Enable authentication (mTLS or SASL) for all connections.
- [ ] Enable authorization (ACLs at minimum).
- [ ] Restrict ZooKeeper/KRaft access to broker nodes only. ZooKeeper in particular is a treasure trove of cluster metadata and historically has had minimal authentication.
- [ ] Disable auto-topic creation. `auto.create.topics.enable=false`. A producer that can create arbitrary topics is a producer that can create a topic named `admin-commands` and confuse your monitoring.
- [ ] Set resource limits: `message.max.bytes`, `max.request.size`, `quota.producer.default`, `quota.consumer.default`. Prevent any single client from monopolizing the cluster.
- [ ] Run the broker process as a non-root user.
- [ ] Enable JMX authentication if JMX is exposed. Unauthenticated JMX can be used for remote code execution.
- [ ] Keep the broker software up to date. CVEs happen.

---

## Common Security Anti-Patterns

### Anti-Pattern 1: "It's on the internal network"

The internal network is not a security boundary. It's a speed bump at best. Internal networks get compromised. Employees go rogue. Contractors have access. That one legacy server running Windows Server 2008 in the corner? It has flat network access to your Kafka cluster.

**Fix:** Zero trust. Authenticate and authorize every connection, regardless of network location.

### Anti-Pattern 2: Shared credentials across services

All services use the same username/password or the same TLS certificate. If one service is compromised, the attacker has the credentials of every service.

**Fix:** Unique credentials per service. This is what mTLS with per-service certificates gives you automatically.

### Anti-Pattern 3: Overly broad topic access

Every service can read from and write to every topic. This is the default configuration for most brokers, and it is exactly as secure as leaving your front door open because you live in a nice neighborhood.

**Fix:** Deny by default. Grant the minimum required access per service.

### Anti-Pattern 4: PII in plaintext everywhere

PII is in events, in consumer databases, in logs, in monitoring dashboards, in Slack alerts that say "Order from alice@example.com failed." Every copy is a liability.

**Fix:** Encrypt PII at the source. Mask PII in logs and alerts. Use opaque identifiers where possible.

### Anti-Pattern 5: No schema validation

Any producer can send any bytes to any topic. A bug in one producer sends garbage data that crashes three consumers and corrupts a database.

**Fix:** Mandatory schema validation at the serialization layer, ideally enforced by the broker.

### Anti-Pattern 6: Secrets in event payloads

API keys, tokens, passwords embedded in events because "the downstream service needs them to call the API." Congratulations, your secrets are now stored in an append-only log, replicated across three broker nodes, retained for seven days, and readable by every consumer of that topic.

**Fix:** Never put secrets in events. Use a secrets manager. Pass references, not values.

### Anti-Pattern 7: Ignoring the control plane

All security focus is on the data plane (events in topics) while the control plane (topic management, ACL management, schema management, consumer group management) is wide open.

**Fix:** Secure the control plane at least as rigorously as the data plane. Ideally more so.

---

## Summary

Security in event-driven systems is not a feature you bolt on after the architecture is built. It's a set of constraints that must inform the architecture from day one. The expanded attack surface — broker, producers, consumers, network, storage, schema registry — demands a comprehensive approach:

1. **Authenticate** every connection with mTLS or SASL.
2. **Authorize** every operation with ACLs or RBAC.
3. **Encrypt** in transit with TLS and at rest with disk or application-level encryption.
4. **Protect PII** with field-level encryption and crypto-shredding for deletion compliance.
5. **Validate schemas** to prevent malformed or malicious events.
6. **Audit** everything, but audit metadata, not payloads.
7. **Harden** brokers, registries, and the network.

The best event-driven security is invisible to developers who are doing the right thing and an impenetrable wall to everyone else. Achieving that is hard work. But the alternative — discovering your GDPR exposure when a regulator asks for proof of deletion from your immutable event log — is harder.
