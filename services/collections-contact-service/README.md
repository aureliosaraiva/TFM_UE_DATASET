# Collections Contact Service

## Overview

The collections-contact-service handles all outbound contact attempts with customers who have overdue payments. It supports phone calls, WhatsApp messages, and email as contact channels. The service doesn't store customer PII itself -- it pulls contact details from customer-service at the time of each attempt.

This service sits at the front of the collections pipeline. When dunning-service kicks off a dunning cycle, this service picks up the event and starts reaching out to the customer. Every contact attempt, successful or not, gets recorded and published as an event so downstream services (like negotiation-engine-service) can act on it.

**Important**: This service enforces contact frequency limits. Brazilian consumer protection regulations (CDC) and Central Bank rules limit how often and when you can contact a debtor. The `contact_frequency_limits` table holds these rules. If you're wondering why a contact attempt was rejected, check there first.

## Getting Started

### Prerequisites

- JDK 17+
- PostgreSQL 14+ (schema: `collections_contact` in `collections_db`)
- Kafka (for event consumption and publishing)
- Access to customer-service (REST)

### Running Locally

```bash
./gradlew bootRun --args='--spring.profiles.active=local'
```

For local development, the service uses a mock telephony provider and a local SMTP server. WhatsApp is stubbed out entirely -- you can't test real WhatsApp messages locally due to Meta's sandbox restrictions.

### Database Setup

```bash
./gradlew flywayMigrate
```

Migrations live in `src/main/resources/db/migration/`. The schema is straightforward: `contact_attempts` is the main table, with foreign keys to `contact_channels` and `contact_outcomes` reference tables.

### Environment Variables

| Variable | Description | Default |
|---|---|---|
| `CUSTOMER_SERVICE_URL` | Base URL for customer-service | `http://localhost:8080` |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka broker addresses | `localhost:9092` |
| `TELEPHONY_PROVIDER_API_KEY` | API key for phone call provider | - |
| `WHATSAPP_API_TOKEN` | WhatsApp Business API token | - |
| `SMTP_HOST` | SMTP server for email contacts | `localhost` |
| `CONTACT_FREQUENCY_CHECK_ENABLED` | Enable/disable frequency limits | `true` |

## Architecture

The service follows a standard Spring Boot architecture with a domain-driven layout:

- **Controllers**: REST endpoints for manual contact attempts and history queries
- **Services**: Business logic for channel selection, frequency validation, contact execution
- **Repositories**: JPA repositories for contact attempt persistence
- **Event Handlers**: Kafka consumers for `dunning_started` and `overdue_installment`
- **Channel Adapters**: Abstraction layer over phone, WhatsApp, and email providers

Channel adapters implement a common `ContactChannelAdapter` interface, making it relatively easy to add new channels. We considered adding SMS as a fourth channel in Q3 2025 but decided against it due to low open rates compared to WhatsApp.

### Tribal Knowledge

- The `contact_outcomes` table has a `requires_followup` boolean. If true, the service automatically schedules a retry. The retry delay is configurable per channel in `application.yml` under `collections.retry-delays`.
- When the telephony provider returns a `BUSY` status, we wait 30 minutes before retrying. This is hardcoded in `PhoneChannelAdapter` -- probably should be configurable but hasn't caused issues yet.
- The WhatsApp integration uses template messages only (HSM templates). Free-form messaging requires the customer to have messaged us first within 24 hours.

## API Reference

### POST /api/v1/contacts/attempt

Records a contact attempt. Can be called by operators or triggered internally by event handlers.

```json
{
  "customer_id": "uuid",
  "channel": "PHONE | WHATSAPP | EMAIL",
  "outcome": "CONNECTED | NO_ANSWER | BUSY | INVALID_NUMBER | PROMISE_TO_PAY | REFUSED | VOICEMAIL",
  "notes": "Optional operator notes",
  "operator_id": "uuid (optional, null for automated attempts)"
}
```

Returns `201 Created` with the created `ContactAttempt` record.

### GET /api/v1/contacts/{customer_id}/history

Returns paginated contact history for a customer. Supports query params:
- `channel` -- filter by channel
- `from` / `to` -- date range filter
- `page`, `size` -- pagination (default size: 20)

## Configuration

Key configuration in `application.yml`:

```yaml
collections:
  retry-delays:
    PHONE: 1800       # 30 min in seconds
    WHATSAPP: 3600    # 1 hour
    EMAIL: 86400      # 24 hours
  frequency-limits:
    max-daily-attempts: 3
    max-weekly-attempts: 10
    quiet-hours-start: "20:00"
    quiet-hours-end: "08:00"
    quiet-hours-timezone: "America/Sao_Paulo"
```

Quiet hours are enforced strictly. Any attempt scheduled during quiet hours gets pushed to the next available window.

## Deployment

Deployed to Kubernetes via Helm chart. The service runs with 2-4 replicas depending on load. Contact attempts are idempotent (deduplication by customer_id + channel + 5-minute window), so scaling is safe.

Health check: `GET /actuator/health`
Readiness: `GET /actuator/health/readiness`

## Monitoring

- **Grafana dashboard**: "Collections Contact Overview" shows real-time contact attempt rates, success rates by channel, and time-of-day heatmaps
- **Key alert**: If contact success rate drops below 30% for any single channel over a 1-hour window, an alert fires in PagerDuty
- **Logs**: Structured JSON logs, search by `customer_id` or `attempt_id` in Kibana

## Known Issues

1. **WhatsApp rate limiting**: Meta's API occasionally throttles us during peak hours (10-12 AM). We have a basic retry with exponential backoff, but during heavy dunning cycles it can cause delays.
2. **Phone number validation**: We trust the phone numbers from customer-service without re-validating. If a number is invalid, we waste an attempt. There's a backlog item to add pre-validation.
3. **Timezone handling**: Quiet hours use the customer's registered timezone from customer-service. If that field is null, we default to `America/Sao_Paulo`, which might be wrong for customers in other timezones.
