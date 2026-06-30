# Lead Intake Service

## Overview

The Lead Intake Service is the front door of CredityFlow's onboarding pipeline. Every potential customer — whether they come through the website, mobile app, or one of our partner integrations — enters the system through this service. It captures the initial contact details, validates them, de-duplicates where possible, and fires off a `lead_created` event so that downstream services (primarily the Eligibility Service) can start doing their thing.

This service was one of the first to be extracted from the original monolith back in early 2024. At the time, lead capture was a controller buried inside the main application, and partner integrations were bolted on as afterthoughts. The extraction gave us the chance to properly model leads as first-class entities and build a clean partner API layer from scratch.

**Tech stack:** Kotlin / Spring Boot, PostgreSQL (`lead_intake_db`).

## Getting Started

### Prerequisites

- JDK 17+
- Docker and Docker Compose (for local PostgreSQL and Kafka)
- Access to the `auth-service` instance (or its mock — see below)

### Running Locally

```bash
# Start dependencies
docker-compose up -d postgres kafka

# Run the service
./gradlew bootRun --args='--spring.profiles.active=local'
```

For partner API testing without spinning up the full auth-service, set `auth.mock.enabled=true` in `application-local.yml`. This accepts any Bearer token — obviously never enable this outside local dev.

### Running Tests

```bash
./gradlew test           # Unit tests
./gradlew integrationTest # Integration tests (needs Docker)
```

Integration tests use Testcontainers for PostgreSQL and an embedded Kafka broker.

## Architecture

The service follows a standard hexagonal architecture pattern:

- **API layer** — Spring MVC controllers handling REST endpoints
- **Domain layer** — Lead, LeadSource, and PartnerChannel entities with associated business rules
- **Infrastructure layer** — JPA repositories, Kafka producer, auth-service client

Incoming leads are validated, persisted inside a database transaction, and then the `lead_created` event is published via a transactional outbox. We use a scheduled poller that reads unpublished events from the outbox table and pushes them to Kafka. This guarantees at-least-once delivery even if Kafka is temporarily unavailable.

### Why PostgreSQL?

We considered MongoDB early on since leads are somewhat document-like, but the need for strong transactional guarantees (especially the outbox pattern) and the team's existing PostgreSQL expertise made it the obvious choice. The schema is straightforward — three main tables plus the outbox.

## API Reference

### POST /api/v1/leads

Create a new lead.

**Request body:**
```json
{
  "fullName": "Maria Silva",
  "email": "maria@example.com",
  "phone": "+5511999998888",
  "cpf": "123.456.789-00",
  "source": "WEBSITE",
  "campaignId": "summer-2025",
  "partnerChannelId": null
}
```

**Response (201):**
```json
{
  "id": "lead-uuid-here",
  "status": "NEW",
  "createdAt": "2025-06-15T10:30:00Z"
}
```

Partner requests must include a `Authorization: Bearer <token>` header. The token is validated against auth-service. If the partner channel has a `partnerChannelId`, it must match a registered PartnerChannel record.

### GET /api/v1/leads/{id}

Returns full lead details. Used primarily by the eligibility-service and the customer-portal-bff.

### GET /api/v1/leads?status=X

Query leads by status. Supports pagination (`page`, `size`) and sorting (`sort`). Valid statuses: `NEW`, `QUALIFIED`, `DISQUALIFIED`, `CONVERTED`, `EXPIRED`.

## Configuration

Key configuration properties (set via environment variables or `application.yml`):

| Property | Description | Default |
|---|---|---|
| `spring.datasource.url` | PostgreSQL connection string | `jdbc:postgresql://localhost:5432/lead_intake_db` |
| `kafka.bootstrap-servers` | Kafka broker addresses | `localhost:9092` |
| `kafka.topic.lead-created` | Topic for lead_created events | `onboarding.lead.created` |
| `auth-service.base-url` | URL of the auth-service | `http://auth-service:8080` |
| `auth.mock.enabled` | Skip real auth validation (local dev only) | `false` |
| `lead.duplicate-check.window-hours` | Time window for duplicate detection | `72` |

## Deployment

The service is deployed as a container on our Kubernetes cluster. Helm chart lives in the `infra/charts/lead-intake-service` directory of the platform repo.

- **Replicas:** 2 in staging, 3 in production
- **Resource requests:** 512Mi memory, 250m CPU
- **Health checks:** `/actuator/health` (liveness and readiness)
- **Secrets:** Database credentials and auth-service client secret are injected via Kubernetes secrets

The service is stateless, so horizontal scaling is straightforward. The only consideration is the outbox poller — we use a database advisory lock to ensure only one instance polls at a time.

## Monitoring

### Key Metrics

- `leads_created_total` — Counter of successfully created leads, labeled by source channel. This is the primary business metric.
- `lead_submission_latency_seconds` — Histogram of end-to-end lead creation time. P99 should stay under 500ms.
- `partner_auth_failures_total` — Spikes here usually mean a partner rotated their credentials without telling us.
- `duplicate_leads_detected_total` — If this suddenly increases, check whether a partner is re-submitting batches.

### Alerts

- **Lead creation error rate > 5%** — Pages the on-call. Usually means a database issue.
- **Partner auth failure spike** — Slack notification to the integrations channel.
- **Outbox lag > 60 seconds** — Means events are not being published to Kafka in a timely manner.

### Dashboards

Two Grafana dashboards are set up:
1. **Lead Intake Overview** — Volume by channel, success/failure rates, latency percentiles
2. **Partner Channel Health** — Per-partner throughput, error rates, and auth failure trends

## Known Issues

1. **Duplicate detection is not perfect.** We match on CPF and email, but typos in email addresses can result in duplicates. There is a backlog item to add fuzzy matching, but it has not been prioritized yet.

2. **Partner rate limiting is application-level only.** If a partner hammers us, the rate limiter protects the service, but the requests still hit the load balancer and consume network resources. Moving this to the API gateway is planned.

3. **The outbox poller can lag under heavy load.** During peak campaign periods (e.g., Black Friday), the poller interval may need to be reduced from 5 seconds to 1 second. There is a feature flag for this: `outbox.poll-interval-ms`.

4. **CPF validation is Brazilian-only.** The service currently only supports Brazilian CPF numbers. If CredityFlow expands to other countries, this validation logic will need to be abstracted.
