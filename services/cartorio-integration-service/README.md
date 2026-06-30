# cartorio-integration-service

## Overview

The cartorio-integration-service manages property lien registration with Brazilian cartorios through the Cartorio Digital platform. Unlike vehicle registrations (which are near-instant), property lien registration is an asynchronous process that typically takes 3-15 business days. The service handles submission, polling for status updates, and managing the "exigencia" workflow where a cartorio requests additional documentation or corrections.

**Service ID:** svc-33
**Domain:** Registry
**Owner:** team-registry
**Tech Stack:** Kotlin / Spring Boot / PostgreSQL

## Getting Started

### Prerequisites

- JDK 17+
- Docker and Docker Compose (for local PostgreSQL, Kafka, and Cartorio Digital mock)
- Gradle 8+

### Running Locally

```bash
# Start dependencies (includes Cartorio Digital mock server)
docker-compose up -d postgres kafka cartorio-mock

# Run database migrations
./gradlew flywayMigrate

# Start the service
./gradlew bootRun --args='--spring.profiles.active=local'
```

The service will start on `http://localhost:8082`. Health check is available at `/actuator/health`.

### Running Tests

```bash
./gradlew test              # Unit tests
./gradlew integrationTest   # Integration tests with Cartorio mock
```

## Architecture

The service is designed around long-running workflows. A scheduled polling job checks Cartorio Digital every 4 hours for status updates on pending registrations. The domain model explicitly handles the "exigencia" state, where a cartorio raises additional requirements.

Key components:
- **EventConsumer** -- Listens for `registry_requested` events filtered for PROPERTY collateral.
- **CartorioRegistrationService** -- Core domain logic for property lien registration.
- **CartorioDigitalClient** -- HTTP client with OAuth2 authentication for Cartorio Digital API calls.
- **StatusPollingScheduler** -- Scheduled job that polls for status updates on pending registrations.
- **CartorioController** -- REST API for manual registration and status queries.

See [architecture.md](architecture.md) for component diagrams and data flow details.

## API Reference

### POST /api/v1/cartorio/register
Submit a property lien registration.

**Request Body:**
```json
{
  "registration_request_id": "uuid",
  "property": {
    "matricula_number": "string",
    "cartorio_code": "string",
    "comarca": "string",
    "state_code": "SP"
  },
  "lien": {
    "contract_id": "uuid",
    "lien_type": "ALIENACAO_FIDUCIARIA",
    "creditor_cnpj": "string",
    "amount": 500000.00
  }
}
```

**Response:** `202 Accepted` with the CartorioRegistration object. Registration processing is asynchronous.

### GET /api/v1/cartorio/status/{id}
Retrieve the current registration status.

**Response:** `200 OK` with CartorioRegistration including protocol number and any exigencia details.

All endpoints require a valid Bearer token in the `Authorization` header.

## Configuration

| Property | Description | Default |
|---|---|---|
| `spring.datasource.url` | PostgreSQL connection URL | `jdbc:postgresql://localhost:5432/registry_db` |
| `spring.kafka.bootstrap-servers` | Kafka broker addresses | `localhost:9092` |
| `app.cartorio.api-url` | Cartorio Digital API base URL | `https://api.cartoriodigital.com.br` |
| `app.cartorio.oauth2.client-id` | OAuth2 client ID | (from secrets) |
| `app.cartorio.oauth2.client-secret` | OAuth2 client secret | (from secrets) |
| `app.cartorio.polling.interval-hours` | Status polling interval | `4` |
| `app.cartorio.polling.max-age-days` | Stop polling after N days | `30` |

## Deployment

- **Docker image:** `credityflow/cartorio-integration-service`
- **Kubernetes namespace:** `registry`
- **Replicas:** 2 (production)
- **Resource limits:** 512Mi memory, 500m CPU
- **Secrets:** OAuth2 credentials are mounted from Kubernetes secrets.
- **CronJob:** Status polling runs as an in-process scheduled task (not a separate CronJob).

## Monitoring

- **Health check:** `GET /actuator/health` (includes Cartorio Digital connectivity check)
- **Metrics:** `GET /actuator/prometheus`
- **Key metrics:** `cartorio_registrations_total`, `cartorio_processing_days`, `cartorio_exigencias_total`
- **Alerts:**
  - `cartorio_api_unavailable` -- critical when Cartorio Digital API errors exceed 50% in 15 minutes.
  - `registration_stale` -- warning when a registration stays IN_ANALYSIS for over 20 business days.
  - `exigencia_unresolved` -- warning when an exigencia goes unresolved for over 5 business days.
- **Logs:** JSON structured logs with `trace_id`, `registration_id`, and `matricula_number` for correlation.

## Known Issues

- Cartorio Digital scheduled maintenance on weekends can cause polling failures; these are expected and retried automatically.
- Exigencia resolution requires manual intervention by the operations team. There is no automated resolution path.
- Some cartorios are slower than others; the 3-15 business day range is an estimate. Outliers of 30+ days have been observed.
- The polling job processes all pending registrations sequentially. For high volumes, this may need to be parallelized.
