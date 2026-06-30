# dunning-service

> **svc-39** | Collections | team-collections | Production

## Overview

When a borrower misses a payment, the dunning-service steps in. It runs automated contact campaigns across SMS, email, and push notifications, following carefully sequenced steps designed to recover the debt before it escalates to legal action. Think of it as the friendly-but-persistent reminder system for CredityFlow's collections pipeline.

The service reacts to `overdue_installment` events from billing-service and orchestrates multi-step dunning sequences. Operations staff can pause campaigns manually when needed (for example, if a borrower calls in to negotiate directly).

## Getting Started

**Requirements:** JDK 17+, Docker Compose, Gradle 8.x

```bash
docker-compose up -d postgres kafka    # infrastructure
./gradlew bootRun -Dspring.profiles.active=local
```

Hit `http://localhost:8081/actuator/health` to confirm the service is running.

### Key Environment Variables

| Variable | Purpose | Default |
|---|---|---|
| `SPRING_DATASOURCE_URL` | JDBC connection to dunning_db | `jdbc:postgresql://localhost:5432/dunning_db` |
| `SPRING_KAFKA_BOOTSTRAP_SERVERS` | Kafka brokers | `localhost:9092` |
| `DUNNING_SMS_GATEWAY_URL` | SMS provider endpoint | -- |
| `DUNNING_EMAIL_PROVIDER_URL` | Email provider endpoint | -- |
| `DUNNING_QUIET_HOURS_START` | Start of quiet hours (local time) | `21:00` |
| `DUNNING_QUIET_HOURS_END` | End of quiet hours (local time) | `07:00` |

## Architecture

The dunning-service is event-driven at its core. A Kafka consumer picks up `overdue_installment` events and triggers the campaign creation logic. Internally, the service uses a **step scheduler** that evaluates pending `DunningStep` records on a fixed interval and dispatches contact attempts through the appropriate channel adapter (SMS, email, or push).

Three main entities drive the domain:

- **DunningCampaign** -- lifecycle wrapper for the entire contact sequence against one overdue installment.
- **DunningStep** -- individual step in the sequence (channel, delay, template).
- **ContactAttempt** -- record of each outbound message with delivery tracking.

See [architecture.md](./architecture.md) for component diagrams and design decisions.

## API Reference

### GET /api/v1/dunning/campaigns

List campaigns with optional filters: `status` (ACTIVE, PAUSED, COMPLETED), `borrower_id`, `page`, `size`.

```json
{
  "content": [
    {
      "campaign_id": "camp-001",
      "installment_id": "inst-456",
      "status": "ACTIVE",
      "current_step": 2,
      "total_steps": 5,
      "started_at": "2026-03-20T08:00:00Z"
    }
  ]
}
```

### POST /api/v1/dunning/campaigns/{id}/pause

Pauses an active campaign. Returns `409 Conflict` if the campaign is already paused or completed.

```json
{
  "campaign_id": "camp-001",
  "status": "PAUSED",
  "paused_at": "2026-03-29T14:30:00Z"
}
```

## Configuration

Campaign sequence templates are defined in `src/main/resources/templates/` as YAML files. Each template specifies the steps, delays, and channel for a dunning sequence. Example:

```yaml
template_id: standard-3step
steps:
  - day: 1
    channel: SMS
    template: sms-reminder-gentle
  - day: 5
    channel: EMAIL
    template: email-reminder-formal
  - day: 10
    channel: PUSH
    template: push-final-warning
```

Quiet-hour enforcement and contact frequency caps are configured via application properties. The service respects a maximum of 3 contact attempts per borrower per week by default.

## Deployment

```bash
./gradlew bootBuildImage --imageName=credityflow/dunning-service:latest

helm upgrade --install dunning-service deploy/helm \
  --namespace collections \
  --values deploy/helm/values-staging.yaml
```

Flyway handles database migrations. Channel provider credentials are mounted as Kubernetes secrets.

## Monitoring

Core metrics exposed at `/actuator/prometheus`:

- `dunning_campaigns_started_total` -- new campaigns launched
- `dunning_contact_attempts_total` -- outbound messages sent (labeled by channel)
- `dunning_contact_delivery_success_rate` -- successful delivery ratio
- `dunning_campaigns_paused_total` -- manual pauses by operations

**Alerts:**
- *LowDeliveryRate* -- delivery success drops below 85%, likely indicating a channel provider issue.
- *CampaignBacklog* -- more than 10,000 pending steps, suggesting the step scheduler is falling behind.

## Known Issues

- **Timezone gaps:** Quiet-hour enforcement depends on borrower timezone from customer-service. If that data is missing, the service defaults to UTC, which may result in contacts during local quiet hours.
- **SMS rate limits:** During high-volume overdue waves, the SMS gateway may throttle requests. The service retries with exponential backoff, but campaigns can fall behind schedule.
- **Duplicate campaigns:** If the `overdue_installment` event is delivered twice, a duplicate campaign may be created. A deduplication window (configurable, default 1 hour) mitigates this but does not eliminate it entirely.
