# Notification Service

## Service Identity

- **ID:** svc-49
- **Domain:** Notifications
- **Team:** team-notifications
- **Stack:** Kotlin / Spring Boot
- **Database:** PostgreSQL (`notification_db`)

## Overview

The Notification Service is CredityFlow's multi-channel messaging hub. Whenever the platform needs to communicate with a customer -- whether it is a proposal status update, a payment reminder, a document request, or a fraud alert -- this service handles the delivery.

It supports four delivery channels:

| Channel | Provider | Use Cases |
|---------|----------|-----------|
| **Email** | AWS SES | Formal communications, contracts, statements |
| **SMS** | AWS SNS | OTP codes, payment reminders, urgent alerts |
| **Push** | Firebase Cloud Messaging | Real-time status updates in the mobile app |
| **WhatsApp** | Twilio | Customer engagement, document follow-ups |

The service consumes **25+ event types** from across the platform. Each event is mapped to one or more notification templates (managed by the companion `notification-template-service`), rendered with context data, and dispatched through the appropriate channel based on customer preferences and notification type configuration.

## Getting Started

Prerequisites: JDK 17+, PostgreSQL, Docker Compose, Gradle.

```bash
docker compose up -d postgres

./gradlew bootRun -Dspring.profiles.active=local
```

Port: **8049**

In the `local` profile, all channel providers are replaced with console loggers -- no real messages are sent. You can see rendered notifications in the application logs.

```bash
./gradlew test                    # unit tests
./gradlew integrationTest         # requires postgres
```

## Architecture

```
  25+ event topics (Kafka)
         |
         v
  +------+------+     +---------------------------+
  | Event       |     | notification-template-svc |
  | Router      +---->| (renders templates)       |
  +------+------+     +---------------------------+
         |
         v
  +------+------+
  | Channel     |
  | Dispatcher  |
  +--+--+--+---+
     |  |  |  |
     v  v  v  v
   SES SNS FCM Twilio
```

### Processing Pipeline

1. **Ingest.** Kafka consumer receives a domain event (e.g., `proposal_approved`).
2. **Route.** The event router looks up which notification types are triggered by this event and which channels to use.
3. **Render.** For each notification, the service calls `notification-template-service` to render the message body with event data.
4. **Dispatch.** The channel dispatcher sends the rendered message through the appropriate provider.
5. **Record.** The outcome (success/failure, provider message ID, timestamps) is persisted to `notification_db`.
6. **Publish.** `notification_requested` is emitted when processing begins; `notification_sent` is emitted on successful delivery (or `notification_failed` on terminal failure).

### Retry Strategy

Failed deliveries are retried with exponential backoff (1min, 5min, 30min) up to 3 attempts. After exhausting retries, the notification is marked as `FAILED` and an alert is raised.

## API Reference

### Send Notification (ad hoc)

```http
POST /api/v1/notifications/send
Content-Type: application/json

{
  "customer_id": "cust-123",
  "template_id": "payment_reminder",
  "channel": "SMS",
  "variables": {
    "amount": "R$ 1.250,00",
    "due_date": "2026-04-05"
  }
}
```

Returns `202 Accepted` with a notification ID. Most notifications are triggered by events, but this endpoint allows manual or system-initiated sends.

### Get Notification Status

```http
GET /api/v1/notifications/{id}/status
```

Returns delivery status, timestamps, provider response, and retry history.

### Published Events

| Event | Description |
|-------|-------------|
| `notification_requested` | Notification processing has started |
| `notification_sent` | Successfully delivered to the provider |

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `NOTIFICATION_DB_URL` | JDBC connection string | `jdbc:postgresql://localhost:5432/notification_db` |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka broker list | `localhost:9092` |
| `TEMPLATE_SERVICE_URL` | notification-template-service URL | `http://notification-template-service:8050` |
| `AWS_SES_REGION` | AWS region for SES | `us-east-1` |
| `AWS_SES_FROM_ADDRESS` | Default sender email | `noreply@credityflow.com` |
| `AWS_SNS_REGION` | AWS region for SNS | `us-east-1` |
| `FIREBASE_CREDENTIALS_PATH` | Path to FCM service account JSON | `/secrets/firebase.json` |
| `TWILIO_ACCOUNT_SID` | Twilio account SID | -- |
| `TWILIO_AUTH_TOKEN` | Twilio auth token | -- |
| `TWILIO_WHATSAPP_FROM` | Twilio WhatsApp sender number | -- |
| `MAX_RETRY_ATTEMPTS` | Max delivery retry count | `3` |
| `CHANNEL_CONSOLE_MODE` | Log instead of sending (for local dev) | `false` |

## Deployment

AWS EKS, `notifications` namespace.

```bash
helm upgrade --install notification-service infra/helm/notification-service \
  --namespace notifications -f values-production.yaml
```

Production:
- 3 replicas (high throughput, especially during batch billing cycles)
- 512Mi-1Gi memory
- 500m-1000m CPU
- Secrets (Twilio, Firebase) mounted via AWS Secrets Manager CSI

Scaling note: during end-of-month billing runs, notification volume can spike 10x. The HPA should be configured aggressively (target 50% CPU) to absorb these bursts.

## Monitoring

**Health:** `GET /actuator/health` (Postgres, Kafka, SES, SNS, FCM, Twilio)

**Metrics:**

| Metric | Type |
|--------|------|
| `notifications.sent_total` | Counter by channel and status |
| `notifications.delivery_latency_seconds` | Histogram by channel |
| `notifications.retry_total` | Counter by channel |
| `notifications.failed_total` | Counter by channel and error type |
| `notifications.kafka_consumer_lag` | Gauge |

**Dashboard:** Grafana > Notifications > Service Overview

**Alerts:**
- Delivery failure rate > 5% on any channel for 10 minutes
- Kafka consumer lag > 5000
- SES bounce rate > 2% (risk of SES sending suspension)
- Twilio API errors sustained for > 5 minutes

## Known Issues

1. **WhatsApp message window.** Twilio's WhatsApp Business API requires that the customer has messaged within the last 24 hours, or the business must use an approved template. The service does not currently track the 24-hour session window, leading to occasional delivery failures for proactive outreach messages.

2. **Template rendering is synchronous.** Each notification makes a synchronous HTTP call to notification-template-service for rendering. Under high load, this becomes a bottleneck. Caching rendered templates or embedding the rendering logic locally is under consideration.

3. **No customer channel preferences.** The channel is determined by notification type configuration, not by customer preference. A customer preference layer (e.g., "prefer WhatsApp over SMS") is planned but not yet built.
