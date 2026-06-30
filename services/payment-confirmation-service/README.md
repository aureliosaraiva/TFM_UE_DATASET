# payment-confirmation-service

## Overview

The payment-confirmation-service closes the disbursement lifecycle by confirming whether bank transfers initiated by CredityFlow have been completed, failed, or rejected. It receives real-time payment confirmation webhooks from BancoParceiro and runs a polling fallback for transfers that do not receive timely webhook notification. A built-in reconciliation engine verifies that confirmed amounts match expected amounts, flagging discrepancies for investigation.

This service is critical for financial accuracy: until a transfer is confirmed, the disbursement is not considered complete and the borrower's loan status remains in a transitional state.

**Service ID:** svc-37
**Domain:** Disbursement
**Owner:** team-disbursement
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

The service will start on `http://localhost:8086`. Health check is available at `/actuator/health`.

For local webhook testing, use a tool like ngrok to expose the webhook endpoint:

```bash
ngrok http 8086
# Configure the ngrok URL as the webhook endpoint in BancoParceiro sandbox
```

### Running Tests

```bash
./gradlew test              # Unit tests
./gradlew integrationTest   # Integration tests (requires Docker)
```

## Architecture

The service is composed of three main components:

- **WebhookReceiver** -- Accepts POST requests from BancoParceiro at `/api/v1/confirmations/webhook`. Validates the HMAC-SHA256 signature, parses the payload, and persists the confirmation. This is the primary confirmation path.
- **PollingScheduler** -- A scheduled job that runs every 5 minutes, querying for transfers that have been in `INITIATED` status for more than 30 minutes without a webhook confirmation. For each pending transfer, it calls the BancoParceiro status API to retrieve the current state.
- **ReconciliationEngine** -- Compares the confirmed amount against the expected transfer amount. Creates a `ReconciliationRecord` with status `MATCHED` or `MISMATCHED`. Mismatches trigger alerts and require manual investigation by the finance team.

After confirmation and reconciliation, the service publishes a `bank_transfer_confirmed` event to the `credityflow.disbursement` Kafka topic, which downstream services (primarily `disbursement-service`) consume to finalize the disbursement order.

See [architecture.md](architecture.md) for component diagrams and data flow details.

## API Reference

### POST /api/v1/confirmations/webhook

Receive a payment confirmation webhook from BancoParceiro.

**Authentication:** HMAC-SHA256 signature in the `X-Webhook-Signature` header. The signature is computed over the raw request body using a shared secret.

**Request Body (from BancoParceiro):**
```json
{
  "protocol": "BP-2024-123456",
  "transfer_id": "uuid",
  "status": "COMPLETED",
  "amount": 150000.00,
  "completed_at": "2024-11-15T14:30:00Z"
}
```

**Response:** `200 OK` -- Acknowledgment. BancoParceiro retries on non-2xx responses.

**Error Responses:**
- `401 Unauthorized` -- Invalid or missing webhook signature.
- `409 Conflict` -- Duplicate confirmation (idempotent, returns 200 on retry).

### GET /api/v1/confirmations/{transfer_id}

Query the confirmation status for a specific bank transfer.

**Authentication:** Bearer token (service-to-service or backoffice).

**Path Parameters:**
- `transfer_id` (UUID) -- The bank transfer ID.

**Response:** `200 OK`
```json
{
  "id": "uuid",
  "transfer_id": "uuid",
  "confirmation_source": "WEBHOOK",
  "bank_protocol": "BP-2024-123456",
  "status": "CONFIRMED",
  "amount_confirmed": 150000.00,
  "confirmed_at": "2024-11-15T14:30:00Z",
  "reconciliation": {
    "status": "MATCHED",
    "expected_amount": 150000.00,
    "confirmed_amount": 150000.00
  }
}
```

**Error Responses:**
- `404 Not Found` -- No confirmation exists for this transfer (may still be pending).

## Configuration

| Property | Description | Default |
|---|---|---|
| `spring.datasource.url` | PostgreSQL connection URL | `jdbc:postgresql://localhost:5432/disbursement_db` |
| `spring.kafka.bootstrap-servers` | Kafka broker addresses | `localhost:9092` |
| `app.webhook.secret` | HMAC-SHA256 shared secret for webhook validation | (required, no default) |
| `app.polling.interval-minutes` | Polling fallback interval | `5` |
| `app.polling.stale-threshold-minutes` | Minutes before a transfer is considered stale | `30` |
| `app.banco-parceiro.status-url` | BancoParceiro transfer status API URL | (required) |
| `app.banco-parceiro.oauth2.client-id` | OAuth2 client ID for BancoParceiro API | (required) |
| `app.banco-parceiro.oauth2.client-secret` | OAuth2 client secret | (required) |

## Deployment

- **Docker image:** `credityflow/payment-confirmation-service`
- **Kubernetes namespace:** `disbursement`
- **Replicas:** 2 (production)
- **Resource limits:** 512Mi memory, 500m CPU
- **Database migrations** run automatically on startup via Flyway.
- **Webhook endpoint** must be accessible from BancoParceiro's IP range. The Kubernetes ingress is configured with an IP whitelist for the webhook path only.
- **Polling scheduler** runs on a single pod using a distributed lock (ShedLock) to prevent duplicate polling across replicas.

## Monitoring

- **Health check:** `GET /actuator/health`
- **Metrics:** `GET /actuator/prometheus`
- **Key metrics:**
  - `payment_confirmations_total` -- Counter by source (WEBHOOK/POLLING) and status.
  - `confirmation_latency_seconds` -- Histogram measuring time from transfer initiation to confirmation.
  - `reconciliation_status_total` -- Counter by reconciliation status (MATCHED/MISMATCHED/PENDING/TIMEOUT).
  - `polling_fallback_triggers_total` -- Counter of transfers confirmed via polling instead of webhook.
- **Reconciliation Dashboard:** A Grafana dashboard (`Disbursement > Payment Reconciliation`) shows real-time confirmation rates, average latency, webhook vs. polling distribution, and any active mismatches. The finance team reviews this dashboard daily.
- **Alerts:**
  - `webhook_signature_failures` -- Critical when webhook validation failures exceed 10% in 10 minutes (possible security issue).
  - `confirmation_delay` -- Warning when transfers go unconfirmed for more than 1 hour.
  - `reconciliation_mismatch` -- Critical on any amount mismatch (requires immediate investigation).
- **Logs:** JSON structured logs with `trace_id`, `transfer_id`, and `confirmation_id`. Financial amounts are logged for audit. Webhook payloads are logged in full for troubleshooting.

## Known Issues

- **Webhook delays:** BancoParceiro occasionally delivers webhooks with delays of up to 20 minutes during peak hours (typically 10:00-12:00 BRT). The polling fallback compensates, but this results in higher polling-sourced confirmations during these windows.
- **Polling interval tuning:** The default 5-minute polling interval is a balance between API rate limits and confirmation latency. Reducing the interval below 3 minutes risks hitting BancoParceiro's rate limit (100 requests/minute). Increasing it above 10 minutes causes unacceptable confirmation delays.
- **Duplicate webhooks:** BancoParceiro may send the same webhook multiple times. The service handles this idempotently using `transfer_id` as a deduplication key, but duplicate payloads are still logged and counted in metrics.
- **Amount reconciliation edge cases:** Currency rounding differences of up to R$0.01 between expected and confirmed amounts are treated as `MATCHED`. Larger discrepancies are flagged as `MISMATCHED` and require manual review.
- **Polling during BancoParceiro maintenance:** The BancoParceiro status API is unavailable during maintenance windows (typically Sunday 00:00-06:00 BRT). Polling attempts during this window fail silently and retry on the next cycle. Transfers initiated just before the maintenance window may not be confirmed until after it ends.
