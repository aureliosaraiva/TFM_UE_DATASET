# disbursement-service

## Overview

The disbursement-service is the entry point for the Disbursement domain in CredityFlow's secured lending platform. Once a lien registration is confirmed (via `registry_completed` event), this service creates a disbursement order and enforces business rules before funds are released. Disbursements exceeding R$500,000 require manual approval from the backoffice team. This service contains sensitive financial data and supports backoffice operations.

**Service ID:** svc-35
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

The service will start on `http://localhost:8084`. Health check is available at `/actuator/health`.

### Running Tests

```bash
./gradlew test              # Unit tests
./gradlew integrationTest   # Integration tests (requires Docker)
```

## Architecture

The service follows a rules-based architecture where disbursement business rules are stored in the database and evaluated at order creation time. This allows the operations team to adjust thresholds and approval requirements without code changes.

Key components:
- **EventConsumer** -- Listens for `registry_completed` events from Kafka.
- **DisbursementService** -- Core domain logic for order creation and rule evaluation.
- **RuleEngine** -- Evaluates configurable disbursement rules against each order.
- **ApprovalService** -- Manages the manual approval workflow for high-value disbursements.
- **DisbursementController** -- REST API for order management and backoffice operations.
- **EventPublisher** -- Publishes `disbursement_requested` events to Kafka.

See [architecture.md](architecture.md) for component diagrams and data flow details.

## API Reference

### POST /api/v1/disbursements
Create a new disbursement order.

**Request Body:**
```json
{
  "contract_id": "uuid",
  "borrower_id": "uuid",
  "amount": 250000.00,
  "currency": "BRL",
  "bank_account": {
    "bank_code": "001",
    "branch": "1234",
    "account_number": "56789-0",
    "account_type": "CORRENTE"
  }
}
```

**Response:** `201 Created` with the DisbursementOrder object.

### GET /api/v1/disbursements/{id}
Retrieve a disbursement order by ID.

**Response:** `200 OK` with the DisbursementOrder object including approval status.

### POST /api/v1/disbursements/{id}/approve
Approve or reject a disbursement order. Requires backoffice role.

**Request Body:**
```json
{
  "decision": "APPROVED",
  "reason": "Verified collateral and documentation"
}
```

**Response:** `200 OK` with the updated DisbursementOrder.

All endpoints require a valid Bearer token. The approve endpoint additionally requires the `backoffice` role.

## Configuration

| Property | Description | Default |
|---|---|---|
| `spring.datasource.url` | PostgreSQL connection URL | `jdbc:postgresql://localhost:5432/disbursement_db` |
| `spring.kafka.bootstrap-servers` | Kafka broker addresses | `localhost:9092` |
| `app.disbursement.approval-threshold` | Amount above which manual approval is required | `500000.00` |
| `app.disbursement.blackout-start` | Start of daily blackout period (no disbursements) | `22:00` |
| `app.disbursement.blackout-end` | End of daily blackout period | `06:00` |

Disbursement rules can also be managed via the `disbursement_rules` database table for runtime configuration.

## Deployment

- **Docker image:** `credityflow/disbursement-service`
- **Kubernetes namespace:** `disbursement`
- **Replicas:** 2 (production)
- **Resource limits:** 512Mi memory, 500m CPU
- **Database migrations** run automatically on startup via Flyway.
- **Access controls:** Strict RBAC; backoffice endpoints require elevated permissions.

## Monitoring

- **Health check:** `GET /actuator/health`
- **Metrics:** `GET /actuator/prometheus`
- **Key metrics:** `disbursement_orders_total`, `disbursement_amount_sum`, `approval_pending_gauge`, `approval_decision_latency_seconds`
- **Alerts:**
  - `approval_queue_backlog` -- warning when more than 20 orders await approval for over 1 hour.
  - `high_rejection_rate` -- warning when rejection rate exceeds 20% in 1 hour.
  - `disbursement_processing_error` -- critical when error rate exceeds 5% in 15 minutes.
- **Logs:** JSON structured logs with `trace_id`, `disbursement_id`, and `contract_id`. Financial amounts are logged for audit but PII is redacted.

## Known Issues

- Manual approval SLA depends on backoffice team availability. Weekend and holiday disbursements may be delayed.
- The blackout period (22:00-06:00) prevents disbursements during off-hours. Orders created during this window are queued until the next business window.
- Disbursement rules are cached in memory with a 5-minute TTL. Rule changes take up to 5 minutes to take effect.
- There is no automated rollback once a disbursement_requested event is published. Cancellation requires manual intervention with the bank-transfer-integration service.
