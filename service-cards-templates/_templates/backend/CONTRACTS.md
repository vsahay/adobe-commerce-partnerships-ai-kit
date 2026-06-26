# Inbound Contracts — {{SERVICE_NAME}}

> Covers all inbound interfaces: HTTP/REST operations and async event contracts.
> Platform / infrastructure concerns live in [`PLATFORM.md`](PLATFORM.md).
>
> **Last Updated:** {{DATE}}

---

## Overview

| Field | Value |
|-------|-------|
| API version(s) | `{{e.g., v1 / v2}}` |
| Stability | `{{STABLE / IN FLUX / INTERNAL}}` |
| Base path | `{{e.g., /api/v1}}` |
| Auth scheme | `{{e.g., Bearer token / API key header / mTLS}}` |
| Auth header(s) | `{{e.g., Authorization: Bearer <token> / x-api-key: <key>}}` |

---

## APIs We Expose

<!-- One `### <METHOD> /<path> — <Purpose>` per operation. The heading slug becomes the anchor referenced by SERVICE_CARD.md → ## 3. Contracts and MODULE_INDEX.md → ## 4. Capability Catalogue. -->

### {{METHOD}} /{{path}} — {{Short Purpose}}

| Field | Value |
|-------|-------|
| Auth required | `{{yes / no / optional}}` |
| Path params | `{{name: type — description}}` |
| Query params | `{{name: type — description}}` |
| Request body | `{{DTO class or "none"}}` |
| Response (2xx) | `{{DTO class or inline shape}}` |
| Error responses | `{{4xx/5xx codes and meaning}}` |
| Delegate | `{{ServiceClass.method()}}` |

<!-- Repeat `### <METHOD> /<path> — <Purpose>` per operation. -->

---

## Async Contracts

<!-- Omit this entire section if the service has no async consumers or publishers. -->

### Events We Consume

<!-- One `#### <topic_name>` per topic. Heading must match the exact topic name — anchors are used by SERVICE_CARD.md → ## 3. Contracts. -->

#### `{{topic_name}}`

| Field | Value |
|-------|-------|
| Topic / queue | `{{topic_name}}` |
| Broker | `{{e.g., Kafka / SQS / RabbitMQ / Pub/Sub}}` |
| Source service | `{{who publishes this}}` |
| Consumer class | `{{ConsumerClassName}}` |
| Deserialization | `{{e.g., JSON → EventDto / Avro / Protobuf}}` |
| Ack posture | `{{e.g., ack on success / ack always / nack on error}}` |
| Error handling | `{{e.g., retry N times → DLQ / log and skip}}` |
| DLQ | `{{topic name or "none"}}` |

**Processing logic:**
<!-- Numbered steps: deserialize → validate → process → ack/nack -->

**Event schema (`{{EventDtoClass}}`):**
```json
{
  "{{field}}": "{{type}}"
}
```

<!-- Repeat `#### <topic_name>` per consumed topic. -->

---

### Events We Publish

<!-- One `#### <topic_name>` per topic. -->

#### `{{topic_name}}`

| Field | Value |
|-------|-------|
| Topic / queue | `{{topic_name}}` |
| Broker | `{{e.g., Kafka / SNS / RabbitMQ / Pub/Sub}}` |
| Publisher class | `{{PublisherClassName.method()}}` |
| Trigger | `{{when is this event emitted}}` |
| Serialization | `{{e.g., JSON / Avro / Protobuf}}` |
| Error handling | `{{e.g., fire-and-forget / retried via outbox}}` |
| Consumers | `{{known downstream consumers}}` |

**Event schema (`{{EventClass}}`):**
```json
{
  "{{field}}": "{{type}}"
}
```

<!-- Repeat `#### <topic_name>` per published topic. -->
