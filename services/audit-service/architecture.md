# Audit Service (svc-48)

## Platform Domain | Kotlin/Spring Boot | Elasticsearch

---

## Mission

Every meaningful action in CredityFlow leaves a trace. The Audit Service is the platform's central audit log -- a high-volume, append-only store of events that captures who did what, when, and on which resource. It serves compliance, security forensics, operational debugging, and regulatory reporting.

The service is designed for write-heavy workloads. It receives events from every other service in the platform and must never become a bottleneck or a single point of failure for the systems it monitors.

## Design Principles

1. **Append-only.** Audit records are immutable. There is no update or delete API.
2. **Fire-and-forget ingestion.** Producers publish audit events asynchronously. A failed audit write must never block a business transaction.
3. **Queryable.** Despite the write-heavy nature, the service supports efficient queries for compliance review, filtered by time range, actor, resource type, action, and free-text search.
4. **Tamper-evident.** Each audit record includes a hash chain linking it to the previous record for the same resource, enabling detection of gaps or modifications at the storage level.

## Components

### Event Ingestion Pipeline

```
All CredityFlow Services
        |
        | AuditEvent (SNS: platform-audit-topic)
        |
        v
SQS: audit-ingestion-queue (high throughput, batch processing)
        |
        v
AuditEventConsumer (batch size: 100, visibility timeout: 60s)
        |
        v
HashChainCalculator (computes chain hash per resource)
        |
        v
Elasticsearch (index: audit-YYYY-MM, 1 primary + 1 replica)
```

The consumer processes events in batches of 100 using Elasticsearch's bulk API. This is critical for throughput -- individual writes would not scale to the event volume (~50k events/hour in production).

### Query API

```
GET /v1/audit/events?resourceType=CONTRACT&resourceId=CTR-2025-00847&from=2025-01-01&to=2025-12-31
GET /v1/audit/events?actorId=USR-1234&action=APPROVE&limit=50
GET /v1/audit/events?search=renegotiation+rejected
```

Queries are served from Elasticsearch with sub-second response times for typical filtered queries. The API supports pagination via search_after (no offset-based pagination -- Elasticsearch limitation for deep pages).

### Retention Manager

A monthly cron job rolls over Elasticsearch indexes. The retention policy:
- Hot tier (current + last 2 months): SSD-backed, full replicas.
- Warm tier (3-12 months): reduced replicas, read-only.
- Cold tier (12-84 months): snapshot to S3, index closed. Restorable on demand.
- Deletion: after 7 years (regulatory minimum for financial records in Brazil).

### Compliance Alert Emitter

The service analyzes ingested events for patterns that warrant attention:
- Unusual volume of admin actions by a single user.
- Access to customer data outside business hours.
- Repeated failed authorization attempts.

When a pattern triggers, a `ComplianceAlertRaised` event is published to SNS, consumed by the case management service.

## Data Model (Elasticsearch Document)

```json
{
  "eventId": "uuid",
  "timestamp": "2025-11-20T14:30:00.123Z",
  "actor": { "id": "USR-1234", "type": "INTERNAL", "ip": "10.0.1.42" },
  "action": "APPROVE_RENEGOTIATION",
  "resourceType": "CONTRACT",
  "resourceId": "CTR-2025-00847",
  "metadata": { "previousStatus": "PENDING", "newStatus": "APPROVED", "reason": "..." },
  "chainHash": "sha256:abc123...",
  "previousChainHash": "sha256:def456...",
  "source": "negotiation-engine-service"
}
```

## Integration Patterns

- **Inbound (write path):** All services publish `AuditEvent` messages to a shared SNS topic. The audit service consumes via a dedicated SQS queue with high throughput settings (long polling, batch receive).
- **Outbound:** `ComplianceAlertRaised` events to SNS. Query API consumed by backoffice-bff.
- **No synchronous dependencies on other services.** The audit service is a leaf node in the dependency graph -- it consumes events but calls nothing.
- **Resilience:** If Elasticsearch is temporarily unavailable, the SQS queue buffers events (configured for 14-day retention). No data loss as long as ES recovers within that window.

## ADR

### ADR-048-001: Elasticsearch over PostgreSQL for audit storage

**Context:** PostgreSQL could store audit records, but the query patterns (free-text search, time-range aggregations, high write throughput) favor a search-optimized store.

**Decision:** Use Elasticsearch as the primary store for audit events. No PostgreSQL in this service.

**Consequences:** Excellent query performance and built-in full-text search. Index lifecycle management (hot/warm/cold) maps naturally to retention requirements. Tradeoff: Elasticsearch is not ACID-compliant -- the hash chain mechanism provides an alternative integrity guarantee. The team must maintain ES cluster health (managed via AWS OpenSearch Service to reduce ops burden).

### ADR-048-002: Hash chain for tamper evidence over database-level controls

**Context:** Regulators expect audit logs to be tamper-evident. Options included database-level write-once controls, blockchain-based logging, or application-level hash chains.

**Decision:** Implement an application-level hash chain: each audit event for a given resource includes a SHA-256 hash of the previous event's hash plus the current event's content. A verification job runs weekly to detect chain breaks.

**Consequences:** Lightweight tamper detection without exotic infrastructure. If a record is deleted or modified, the chain breaks and the weekly verification job alerts the compliance team. Not as strong as a blockchain guarantee, but sufficient for regulatory requirements and vastly simpler to operate.
