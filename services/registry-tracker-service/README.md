# registry-tracker-service

## Overview

The registry-tracker-service is the final piece of the Registry domain, responsible for aggregating registration status updates from both DETRAN and cartorio integrations. It implements retry logic with exponential backoff for failed registrations and emits the definitive `registry_completed` or `registry_failed` events that downstream services (particularly disbursement) depend on to proceed.

**Service ID:** svc-34
**Domain:** Registry
**Owner:** team-registry
**Tech Stack:** Kotlin / Spring Boot / PostgreSQL

## Getting Started

### Prerequisites

- JDK 17+
- Docker and Docker Compose (for local PostgreSQL and Kafka)
- Gradle 8+

### Running Locally

```bash
# Start dependencies
docker-compose up -d postgres kafka

# Run database migrations
./gradlew flywayMigrate

# Start the service
./gradlew bootRun --args='--spring.profiles.active=local'
```

The service will start on `http://localhost:8083`. Health check is available at `/actuator/health`.

### Running Tests

```bash
./gradlew test              # Unit tests
./gradlew integrationTest   # Integration tests (requires Docker)
```

## Architecture

The service acts as a status aggregator and retry coordinator. It consumes completion/failure events from DETRAN and cartorio integrations, updates a unified tracking model, and applies retry logic when registrations fail.

Key components:
- **EventConsumer** -- Listens for `detran_lien_registered` and `cartorio_lien_registered` events.
- **RegistrationTrackingService** -- Core domain logic for status aggregation and retry decisions.
- **RetryScheduler** -- Scheduled job that executes pending retries with exponential backoff.
- **EventPublisher** -- Publishes `registry_completed` and `registry_failed` events.
- **TrackingController** -- REST API for querying registration status and pending registrations.

See [architecture.md](architecture.md) for component diagrams and data flow details.

## API Reference

### GET /api/v1/registrations/{id}/track
Get the tracking status of a specific registration.

**Response:** `200 OK`
```json
{
  "id": "uuid",
  "registration_request_id": "uuid",
  "channel": "DETRAN",
  "status": "COMPLETED",
  "last_updated": "2026-03-29T10:00:00Z",
  "completion_details": "Protocol: ABC123",
  "retry_count": 0
}
```

### GET /api/v1/registrations/pending
List all registrations with pending or retrying status.

**Query Parameters:** `channel` (optional), `page` (default 0), `size` (default 20)

**Response:** `200 OK` with paginated list of RegistrationStatus objects.

All endpoints require a valid Bearer token in the `Authorization` header.

## Configuration

| Property | Description | Default |
|---|---|---|
| `spring.datasource.url` | PostgreSQL connection URL | `jdbc:postgresql://localhost:5432/registry_db` |
| `spring.kafka.bootstrap-servers` | Kafka broker addresses | `localhost:9092` |
| `app.retry.max-attempts` | Maximum retry attempts before marking as failed | `5` |
| `app.retry.initial-delay-seconds` | Initial delay before first retry | `60` |
| `app.retry.multiplier` | Exponential backoff multiplier | `2.0` |
| `app.retry.max-delay-seconds` | Maximum delay between retries | `3600` |
| `app.retry.scheduler-interval-seconds` | How often the retry scheduler runs | `30` |

## Deployment

- **Docker image:** `credityflow/registry-tracker-service`
- **Kubernetes namespace:** `registry`
- **Replicas:** 2 (production)
- **Resource limits:** 512Mi memory, 500m CPU
- **Database migrations** run automatically on startup via Flyway.

## Monitoring

- **Health check:** `GET /actuator/health`
- **Metrics:** `GET /actuator/prometheus`
- **Key metrics:** `registration_completions_total`, `registration_failures_total`, `retry_attempts_total`, `pending_registrations_gauge`
- **Alerts:**
  - `high_failure_rate` -- critical when failure rate exceeds 10% in 1 hour.
  - `max_retries_exceeded` -- warning when any registration reaches the maximum retry limit.
  - `pending_registrations_backlog` -- warning when pending count exceeds 100.
- **Logs:** JSON structured logs with `trace_id` and `registration_id` for correlation.

## Known Issues

- Cartorio registrations naturally inflate the pending count due to their 3-15 business day processing time. The `pending_registrations_backlog` alert threshold should account for this baseline.
- The retry scheduler processes retries sequentially within a single instance. Under high retry volumes, consider increasing replicas or parallelizing the scheduler.
- If both DETRAN and cartorio integrations are down simultaneously, the retry queue can grow rapidly. Monitor `pending_registrations_gauge` during known outages.
- The `registry_failed` event triggers manual escalation; there is no automated recovery path beyond the retry mechanism.
