# Audit Service

**Service ID:** svc-48 | **Domain:** Platform | **Team:** team-platform
**Stack:** Kotlin / Spring Boot | **Data Store:** Elasticsearch (`audit_log_store`)

## Overview

The Audit Service is CredityFlow's central compliance audit trail. It ingests events from every domain in the platform -- loan origination, fraud detection, collections, disbursement, and more -- and persists them in an immutable, searchable log backed by Elasticsearch.

At present the service subscribes to **23+ distinct event types** across all bounded contexts. Every write operation that touches customer data, financial records, or regulatory artifacts produces an audit entry here. The resulting dataset is the primary source of truth for internal compliance reviews, external regulatory audits, and incident forensics.

> **Data Sensitivity:** Audit records contain **PII and financial data** embedded in event payloads. Access is restricted to authorized compliance and platform personnel. All queries are themselves logged.

## Getting Started

### Prerequisites

- JDK 17+
- Docker & Docker Compose (for local Elasticsearch)
- Gradle 8.x (wrapper included)
- Access to the CredityFlow Kafka cluster (or a local substitute)

### Local Development

```bash
# Start Elasticsearch locally
docker compose up -d elasticsearch

# Run the service
./gradlew bootRun --args='--spring.profiles.active=local'
```

The service will be available at `http://localhost:8048`. Elasticsearch is expected on `localhost:9200`.

### Running Tests

```bash
./gradlew test          # unit tests
./gradlew integrationTest   # requires Elasticsearch container
```

## Architecture

The Audit Service follows an event-driven, append-only architecture:

```
 Kafka (23+ event topics)
        |
        v
 +-----------------+
 | Event Consumers |  (one consumer group per topic partition)
 +-----------------+
        |
        v
 +-------------------+
 | Audit Log Writer  |  normalises events into a canonical AuditEntry
 +-------------------+
        |
        v
 +---------------------+        +------------------+
 | Elasticsearch Index |  --->  | REST Query Layer |
 | (audit_log_store)   |        | (read-only API)  |
 +---------------------+        +------------------+
        |
        v
 Kafka (audit_event_logged)
```

Key design decisions:

- **Append-only storage.** Records are never updated or deleted. Index lifecycle management (ILM) handles retention.
- **Canonical schema.** All incoming events are mapped to a unified `AuditEntry` document with fields for actor, action, entity type/ID, timestamp, and a raw payload blob.
- **Outbound event.** After successful persistence the service publishes `audit_event_logged` so downstream consumers (e.g., SIEM integration) can react.

## API Reference

All endpoints require a valid internal JWT with the `audit:read` scope.

### List Audit Events

```
GET /api/v1/audit/events
```

Query parameters: `entity_type`, `entity_id`, `actor_id`, `action`, `from`, `to`, `page`, `size`.

Returns a paginated list of audit entries matching the supplied filters.

### Get Audit Events by Entity

```
GET /api/v1/audit/events/{entity_type}/{entity_id}
```

Returns all audit entries associated with a specific entity (e.g., `proposals/abc-123`). Results are sorted newest-first by default.

### Published Events

| Event | Topic | Description |
|---|---|---|
| `audit_event_logged` | `platform.audit` | Emitted after every successful audit write |

## Configuration

Key environment variables:

| Variable | Description | Default |
|---|---|---|
| `ELASTICSEARCH_HOSTS` | Comma-separated ES node URLs | `http://localhost:9200` |
| `ELASTICSEARCH_INDEX_PREFIX` | Index name prefix | `audit_log` |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka broker addresses | `localhost:9092` |
| `KAFKA_CONSUMER_GROUP` | Consumer group ID | `audit-service` |
| `ILM_RETENTION_DAYS` | Days before audit indices roll into warm/cold | `365` |
| `PII_MASKING_ENABLED` | Mask PII fields in query responses for non-privileged callers | `true` |

Spring profiles: `local`, `staging`, `production`.

## Deployment

The service is deployed as a container on **AWS EKS**. Helm chart lives in `infra/helm/audit-service/`.

```bash
helm upgrade --install audit-service ./infra/helm/audit-service \
  --namespace platform \
  --values values-production.yaml
```

Resource recommendations (production):
- **Replicas:** 3
- **CPU:** 500m request / 1000m limit
- **Memory:** 768Mi request / 1536Mi limit

Rolling deployments are safe; the Kafka consumer group rebalances automatically.

## Monitoring

| Metric | Type | Alert Threshold |
|---|---|---|
| `audit.events.consumed` | Counter | < 100 events/min triggers warning |
| `audit.events.write_latency_ms` | Histogram | p99 > 500ms |
| `audit.elasticsearch.health` | Gauge | cluster status != green |
| `audit.consumer.lag` | Gauge | > 10 000 messages |

Dashboards: Grafana folder **Platform / Audit Service**.

Health check: `GET /actuator/health` (includes Elasticsearch and Kafka connectivity).

## Known Issues

1. **High consumer lag during reindexing.** When Elasticsearch ILM triggers a rollover and force-merge, write latency spikes and consumer lag can grow. This is transient and self-resolving but may trigger false alerts. A circuit breaker for back-pressure is on the roadmap.

2. **PII in raw payloads.** The `raw_payload` field stores the original event JSON, which may contain unmasked PII. The masking layer only applies to query responses, not to the stored document. This is intentional for forensic completeness, but it means Elasticsearch node-level encryption and access controls are critical.

3. **No bulk query endpoint.** Compliance teams occasionally need to export large date ranges. Currently this requires scrolling through paginated results. A bulk export endpoint (possibly backed by an async job) is planned.
