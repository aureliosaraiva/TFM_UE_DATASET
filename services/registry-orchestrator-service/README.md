# registry-orchestrator-service

## Overview

The registry-orchestrator-service is the central coordination point for lien registration within CredityFlow's secured lending platform. When a contract is signed, this service determines whether the collateral requires registration with DETRAN (vehicles) or a cartorio (properties) and routes the request accordingly. It implements the Saga pattern to manage the multi-step, distributed registration workflow, ensuring consistency and traceability throughout the process.

**Service ID:** svc-31
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

The service will start on `http://localhost:8080`. Health check is available at `/actuator/health`.

### Running Tests

```bash
./gradlew test          # Unit tests
./gradlew integrationTest  # Integration tests (requires Docker)
```

## Architecture

The service follows a hexagonal architecture with clear separation between domain logic (routing decisions, Saga management) and infrastructure concerns (Kafka consumers/producers, database access).

Key components:
- **RegistrationController** -- REST API layer for creating and querying registration requests.
- **RegistrationOrchestrator** -- Core domain logic that determines the routing channel based on collateral type.
- **SagaManager** -- Manages Saga state transitions and compensation flows.
- **EventConsumer** -- Listens for `contract_signed` events from Kafka.
- **EventPublisher** -- Publishes `registry_requested` events to Kafka.

See [architecture.md](architecture.md) for component diagrams and data flow details.

## API Reference

### POST /api/v1/registrations
Create a new lien registration request.

**Request Body:**
```json
{
  "contract_id": "uuid",
  "collateral_type": "VEHICLE | PROPERTY",
  "collateral_details": { }
}
```

**Response:** `201 Created` with the created RegistrationRequest.

### GET /api/v1/registrations/{id}
Retrieve a registration request by its ID.

**Response:** `200 OK` with the RegistrationRequest object, including current status and Saga state.

All endpoints require a valid Bearer token in the `Authorization` header.

## Configuration

Key application properties:

| Property | Description | Default |
|---|---|---|
| `spring.datasource.url` | PostgreSQL connection URL | `jdbc:postgresql://localhost:5432/registry_db` |
| `spring.kafka.bootstrap-servers` | Kafka broker addresses | `localhost:9092` |
| `app.saga.timeout-minutes` | Max time before a Saga is considered stuck | `30` |
| `app.routing.default-channel` | Fallback routing channel | `DETRAN` |

Environment-specific configuration is managed via Spring profiles (`local`, `staging`, `production`).

## Deployment

The service is containerized and deployed via the standard CredityFlow CI/CD pipeline.

- **Docker image:** `credityflow/registry-orchestrator-service`
- **Kubernetes namespace:** `registry`
- **Replicas:** 2 (production)
- **Resource limits:** 512Mi memory, 500m CPU
- **Database migrations** run automatically on startup via Flyway.

## Monitoring

- **Health check:** `GET /actuator/health`
- **Metrics:** `GET /actuator/prometheus` (Prometheus format)
- **Key metrics:** `registration_requests_total`, `saga_duration_seconds`, `routing_decisions_total`
- **Alerts:**
  - `saga_stuck` -- warning when a Saga remains in IN_PROGRESS for over 30 minutes.
  - `high_failure_rate` -- critical when registration failure rate exceeds 5% within 15 minutes.
- **Logs:** JSON structured logs with `trace_id`, `span_id`, and `registration_id` for correlation.

## Known Issues

- Saga compensation for partially completed registrations requires manual intervention in edge cases where both downstream services fail simultaneously.
- Routing logic assumes collateral type is always present in the contract_signed event payload; missing data will cause the registration to fail immediately.
- Under high load, Kafka consumer lag may delay registration processing. Monitor `kafka_consumer_lag` metric.
