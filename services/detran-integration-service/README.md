# detran-integration-service

## Overview

The detran-integration-service handles vehicle lien registration (gravame) with Brazilian state-level DETRAN systems. Each state operates its own DETRAN API with varying contracts, authentication mechanisms, and processing times. This service abstracts those differences behind a unified interface, ensuring vehicle liens are registered promptly to protect the lender's collateral interest.

**Service ID:** svc-32
**Domain:** Registry
**Owner:** team-registry
**Tech Stack:** Kotlin / Spring Boot / PostgreSQL

## Getting Started

### Prerequisites

- JDK 17+
- Docker and Docker Compose (for local PostgreSQL, Kafka, and DETRAN mock)
- Gradle 8+
- DETRAN mock certificates (for local/test environments)

### Running Locally

```bash
# Start dependencies (includes DETRAN mock server)
docker-compose up -d postgres kafka detran-mock

# Run database migrations
./gradlew flywayMigrate

# Start the service
./gradlew bootRun --args='--spring.profiles.active=local'
```

The service will start on `http://localhost:8081`. Health check is available at `/actuator/health`.

### Running Tests

```bash
./gradlew test              # Unit tests
./gradlew integrationTest   # Integration tests with DETRAN mock
```

## Architecture

The service uses a strategy pattern to handle state-specific DETRAN API variations. Each supported state has a dedicated adapter implementing a common `DetranRegistrationPort` interface.

Key components:
- **EventConsumer** -- Listens for `registry_requested` events filtered for VEHICLE collateral.
- **DetranRegistrationService** -- Core domain logic for lien registration orchestration.
- **StateAdapterFactory** -- Resolves the correct DETRAN adapter based on the vehicle's state code.
- **DetranApiClient** -- HTTP client with mTLS support for DETRAN API calls.
- **DetranController** -- REST API for manual registration and status queries.

See [architecture.md](architecture.md) for component diagrams and data flow details.

## API Reference

### POST /api/v1/detran/register
Submit a vehicle lien registration to DETRAN.

**Request Body:**
```json
{
  "registration_request_id": "uuid",
  "vehicle": {
    "renavam": "string",
    "chassis_number": "string",
    "license_plate": "string",
    "state_code": "SP"
  },
  "lien": {
    "contract_id": "uuid",
    "creditor_cnpj": "string",
    "amount": 150000.00
  }
}
```

**Response:** `202 Accepted` with the DetranRegistration object.

### GET /api/v1/detran/status/{id}
Retrieve the current registration status.

**Response:** `200 OK` with DetranRegistration including DETRAN protocol number when available.

All endpoints require a valid Bearer token in the `Authorization` header.

## Configuration

| Property | Description | Default |
|---|---|---|
| `spring.datasource.url` | PostgreSQL connection URL | `jdbc:postgresql://localhost:5432/registry_db` |
| `spring.kafka.bootstrap-servers` | Kafka broker addresses | `localhost:9092` |
| `app.detran.timeout-seconds` | HTTP timeout for DETRAN API calls | `30` |
| `app.detran.retry.max-attempts` | Max retry attempts per registration | `3` |
| `app.detran.states.enabled` | Comma-separated list of enabled states | `SP,RJ,MG,PR,RS` |
| `app.detran.mtls.keystore-path` | Path to mTLS keystore | `/certs/detran-keystore.p12` |

State-specific configuration (endpoints, API keys) is managed via `application-{state}.yml` files.

## Deployment

- **Docker image:** `credityflow/detran-integration-service`
- **Kubernetes namespace:** `registry`
- **Replicas:** 2 (production)
- **Resource limits:** 512Mi memory, 500m CPU
- **Secrets:** mTLS certificates and API keys are mounted from Kubernetes secrets.
- **Database migrations** run automatically on startup via Flyway.

## Monitoring

- **Health check:** `GET /actuator/health` (includes DETRAN connectivity checks per state)
- **Metrics:** `GET /actuator/prometheus`
- **Key metrics:** `detran_registrations_total`, `detran_api_latency_seconds`, `detran_api_errors_total` (all labeled by state)
- **Alerts:**
  - `detran_api_down` -- critical when error rate exceeds 50% for any state in 10 minutes.
  - `registration_timeout` -- warning when a registration stays PENDING for over 2 hours.
- **Logs:** JSON structured logs with `trace_id`, `registration_id`, and `renavam` for correlation.

## Known Issues

- DETRAN-SP occasionally returns HTTP 200 with an error payload; the adapter includes special handling for this case.
- Some states (e.g., BA, CE) are not yet integrated and will cause registration failures. Check `app.detran.states.enabled` for the current list.
- mTLS certificate expiration is state-specific and must be tracked manually. A renewal reminder is configured in the team calendar.
- Rate limiting from DETRAN-RJ can cause bursts of 429 errors during peak hours (10:00-12:00 BRT).
