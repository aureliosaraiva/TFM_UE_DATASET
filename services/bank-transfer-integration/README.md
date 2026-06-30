# bank-transfer-integration

## Overview

The bank-transfer-integration service executes fund transfers to borrowers' bank accounts via the BancoParceiro API. It supports both TED (traditional bank transfer) and PIX (instant payment) methods, selecting the most appropriate option based on the transaction characteristics. This service contains sensitive financial data and operates under strict banking hour and regulatory constraints.

**Service ID:** svc-36
**Domain:** Disbursement
**Owner:** team-disbursement
**Tech Stack:** Kotlin / Spring Boot / PostgreSQL

## Getting Started

### Prerequisites

- JDK 17+
- Docker and Docker Compose (for local PostgreSQL, Kafka, and BancoParceiro mock)
- Gradle 8+
- BancoParceiro sandbox credentials (for integration testing)

### Running Locally

```bash
# Start dependencies (includes BancoParceiro mock server)
docker-compose up -d postgres kafka banco-mock

# Run database migrations
./gradlew flywayMigrate

# Start the service
./gradlew bootRun --args='--spring.profiles.active=local'
```

The service will start on `http://localhost:8085`. Health check is available at `/actuator/health`.

### Running Tests

```bash
./gradlew test              # Unit tests
./gradlew integrationTest   # Integration tests with BancoParceiro mock
```

## Architecture

The service uses a strategy pattern for transfer method selection and a resilient HTTP client for BancoParceiro communication. All transfer data is encrypted at rest in the database.

Key components:
- **EventConsumer** -- Listens for `disbursement_requested` events from Kafka.
- **TransferService** -- Core domain logic for transfer execution and method selection.
- **TransferMethodSelector** -- Determines TED vs PIX based on pix_key availability, amount limits, and banking hours.
- **BancoParceiroClient** -- HTTP client with OAuth2 authentication for BancoParceiro API calls.
- **TransferController** -- REST API for manual transfer initiation and status queries.
- **EventPublisher** -- Publishes `bank_transfer_initiated` events to Kafka.

See [architecture.md](architecture.md) for component diagrams and data flow details.

## API Reference

### POST /api/v1/transfers
Initiate a bank transfer.

**Request Body:**
```json
{
  "disbursement_order_id": "uuid",
  "amount": 250000.00,
  "currency": "BRL",
  "recipient": {
    "bank_code": "001",
    "branch": "1234",
    "account_number": "56789-0",
    "account_type": "CORRENTE",
    "holder_cpf_cnpj": "123.456.789-00",
    "holder_name": "John Doe",
    "pix_key": "john@example.com"
  }
}
```

**Response:** `202 Accepted` with the BankTransfer object.

### GET /api/v1/transfers/{id}/status
Check the status of a bank transfer.

**Response:** `200 OK` with BankTransfer including bank protocol number when available.

All endpoints require a valid Bearer token in the `Authorization` header.

## Configuration

| Property | Description | Default |
|---|---|---|
| `spring.datasource.url` | PostgreSQL connection URL | `jdbc:postgresql://localhost:5432/disbursement_db` |
| `spring.kafka.bootstrap-servers` | Kafka broker addresses | `localhost:9092` |
| `app.banco-parceiro.api-url` | BancoParceiro API base URL | `https://api.bancoparceiro.com.br` |
| `app.banco-parceiro.oauth2.client-id` | OAuth2 client ID | (from secrets) |
| `app.banco-parceiro.oauth2.client-secret` | OAuth2 client secret | (from secrets) |
| `app.banco-parceiro.timeout-seconds` | HTTP timeout for API calls | `30` |
| `app.transfer.pix-max-amount` | Maximum amount for PIX transfers | `1000000.00` |
| `app.transfer.ted-start-hour` | TED window start (BRT) | `06:30` |
| `app.transfer.ted-end-hour` | TED window end (BRT) | `17:00` |

## Deployment

- **Docker image:** `credityflow/bank-transfer-integration`
- **Kubernetes namespace:** `disbursement`
- **Replicas:** 2 (production)
- **Resource limits:** 512Mi memory, 500m CPU
- **Egress:** Must use static egress IPs (IP whitelisted with BancoParceiro).
- **Secrets:** OAuth2 credentials and encryption keys are mounted from Kubernetes secrets.
- **Database migrations** run automatically on startup via Flyway.

## Monitoring

- **Health check:** `GET /actuator/health` (includes BancoParceiro connectivity check)
- **Metrics:** `GET /actuator/prometheus`
- **Key metrics:** `bank_transfers_total`, `bank_transfer_amount_sum`, `banco_parceiro_api_latency_seconds`, `banco_parceiro_api_errors_total`
- **Alerts:**
  - `banco_parceiro_api_down` -- critical when error rate exceeds 50% in 10 minutes.
  - `transfer_submission_failure` -- critical when transfer failure rate exceeds 5% in 15 minutes.
  - `high_transfer_latency` -- warning when p99 latency exceeds 30 seconds.
- **Logs:** JSON structured logs with `trace_id`, `transfer_id`, and `disbursement_order_id`. Bank account numbers are partially masked in logs.

## Known Issues

- TED transfers submitted outside banking hours (06:30-17:00 BRT) will be queued by BancoParceiro and processed on the next business day.
- PIX transfers exceeding R$1,000,000 are rejected by Central Bank regulation. The service automatically falls back to TED for these amounts.
- BancoParceiro scheduled maintenance (Sunday 00:00-06:00) causes all transfers to fail during that window. The service queues them for retry.
- Static egress IP requirement means the service cannot be freely migrated between Kubernetes clusters without updating the IP whitelist with BancoParceiro.
