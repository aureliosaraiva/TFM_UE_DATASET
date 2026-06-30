# Case Management Service

`svc-47` -- Backoffice domain -- Owned by team-backoffice

## Overview

When automated decision engines in CredityFlow flag an item for human review, the Case Management Service takes over. It is the system of record for **manual review cases** spanning fraud investigations, failed registry lookups, disputed credit decisions, and collections escalations.

The service listens for domain events that indicate a manual review is needed, creates structured cases with appropriate metadata, routes them to the right review queue, and tracks them through to resolution. It is technology-wise a Kotlin/Spring Boot application backed by PostgreSQL (`case_management_db`).

## Getting Started

**Requirements:** JDK 17+, PostgreSQL 14+, Docker Compose, Gradle.

```bash
# spin up postgres
docker compose up -d postgres

# apply migrations and start
./gradlew bootRun -Dspring.profiles.active=local
```

Service starts on port **8047**. The local profile seeds sample cases for development.

```bash
# unit + integration tests
./gradlew check
```

## Architecture

### Event Flow

```
  fraud_flagged ──────────┐
  registry_failed ────────┤
  collections_contact_ ───┤
    attempted              │
                           v
                 +--------------------+
                 | Case Management    |
                 | Service            |
                 +--------------------+
                   |              |
                   v              v
             case_created    case_resolved
             (event out)     (event out)
                   |
                   v
             +-------------+
             | PostgreSQL   |
             | case_mgmt_db|
             +-------------+
```

### Data Model

A `Case` record contains:

| Field | Description |
|-------|-------------|
| `id` | UUID primary key |
| `type` | `FRAUD`, `REGISTRY`, `CREDIT`, `COLLECTIONS` |
| `status` | `OPEN`, `IN_PROGRESS`, `RESOLVED`, `ESCALATED` |
| `priority` | `LOW`, `MEDIUM`, `HIGH`, `CRITICAL` |
| `source_event` | The originating event type and payload |
| `entity_type` / `entity_id` | The business entity under review (e.g., proposal, customer) |
| `assigned_to` | Agent user ID (nullable until claimed) |
| `resolution` | Outcome details, populated on resolve |
| `created_at` / `updated_at` | Timestamps |

### Consumed Events

| Event | Source | Triggers |
|-------|--------|----------|
| `fraud_flagged` | fraud-decision-service | Creates a FRAUD case |
| `registry_failed` | cartorio-integration-service | Creates a REGISTRY case |
| `collections_contact_attempted` | collections-contact-service | Creates a COLLECTIONS case if max attempts reached |

### Published Events

| Event | When |
|-------|------|
| `case_created` | New case persisted |
| `case_resolved` | Case marked as resolved or escalated |

## API Reference

### Create Case

```http
POST /api/v1/cases
Content-Type: application/json

{
  "type": "FRAUD",
  "priority": "HIGH",
  "entity_type": "proposal",
  "entity_id": "prop-abc-123",
  "description": "Multiple identity documents detected"
}
```

Returns `201 Created` with the full case object. Normally cases are created via events, but this endpoint supports ad-hoc case creation by back-office agents.

### Get Case

```http
GET /api/v1/cases/{id}
```

Returns the case with full history (status transitions, comments, assignment changes).

### Resolve Case

```http
PUT /api/v1/cases/{id}/resolve

{
  "resolution": "APPROVED",
  "comment": "Documents verified manually, proceeding with proposal."
}
```

Transitions the case to `RESOLVED` and publishes `case_resolved`. Valid resolution values depend on the case type.

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `CASE_DB_URL` | JDBC URL for case_management_db | `jdbc:postgresql://localhost:5432/case_management_db` |
| `CASE_DB_USERNAME` | DB user | `case_mgmt` |
| `CASE_DB_PASSWORD` | DB password | -- |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka brokers | `localhost:9092` |
| `KAFKA_CONSUMER_GROUP` | Consumer group | `case-management-service` |
| `AUTO_ASSIGN_ENABLED` | Round-robin auto-assignment of new cases | `false` |
| `CASE_SLA_HOURS_HIGH` | SLA for HIGH priority cases | `4` |
| `CASE_SLA_HOURS_CRITICAL` | SLA for CRITICAL priority cases | `1` |

## Deployment

Helm chart: `infra/helm/case-management-service/`

```bash
helm upgrade --install case-management-service \
  infra/helm/case-management-service \
  --namespace backoffice \
  -f values-production.yaml
```

Production sizing: 2 replicas, 512Mi memory, 500m CPU. Database migrations are handled by Flyway on startup.

## Monitoring

**Health:** `GET /actuator/health` (Postgres + Kafka)

**Metrics:**
- `cases.created_total` -- counter by type and priority
- `cases.resolved_total` -- counter by type and resolution
- `cases.open_count` -- gauge by type (tracks backlog)
- `cases.sla_breached_total` -- counter of cases exceeding their SLA

**Dashboard:** Grafana > Backoffice > Case Management

**Alerts:**
- Open CRITICAL cases > 0 for more than 1 hour
- SLA breach rate > 10% in any rolling 24h window
- Kafka consumer lag > 1000

## Known Issues

1. **No case merging.** When multiple events fire for the same entity (e.g., a proposal triggers both a fraud flag and a registry failure), separate cases are created. Agents must manually cross-reference. A deduplication or merge feature is planned.

2. **Auto-assignment is naive.** The round-robin assigner does not account for agent workload, skills, or shift schedules. It works acceptably for small teams but will need a smarter routing algorithm as the ops team scales.

3. **Event replay creates duplicates.** If upstream events are replayed (e.g., during a Kafka consumer reset), duplicate cases will be created. An idempotency key based on the source event ID is the intended fix.
