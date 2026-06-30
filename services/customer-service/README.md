# customer-service

## Overview

The customer-service is the central authority for customer identity and profile data within CredityFlow. It manages the full lifecycle of customer records -- creation from qualified leads, profile updates, and lookups used by nearly every other service in the platform.

This service stores personally identifiable information (PII) including full name, CPF, email, phone numbers, and physical addresses. Because of this, it carries strict access control and encryption requirements. All PII columns in the database are encrypted at rest via AWS KMS, and access is audited.

The service is owned by **team-onboarding** and sits in the **Onboarding** domain.

## Getting Started

### Prerequisites

- JDK 17+
- Docker and Docker Compose (for local PostgreSQL)
- AWS CLI configured (for KMS integration in non-local profiles)

### Running Locally

```bash
# Start dependencies
docker-compose up -d postgres

# Run the application
./gradlew bootRun --args='--spring.profiles.active=local'
```

The service starts on port `8080` by default. Health check is available at `/actuator/health`.

### Running Tests

```bash
./gradlew test          # Unit tests
./gradlew integrationTest  # Integration tests (requires Docker)
```

## Architecture

The customer-service is a Kotlin/Spring Boot application backed by PostgreSQL. It follows a standard layered architecture: controller -> service -> repository. Domain events are published to SNS topics and consumed from SQS queues.

Key architectural decisions:
- **Event-driven ingestion**: Customer profiles are created reactively from `lead_qualified` events rather than requiring synchronous API calls from the lead pipeline.
- **Optimistic locking**: Profile updates use version-based optimistic locking to handle concurrent modifications from multiple channels.

For more details, see [architecture.md](architecture.md).

## API Reference

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/customers` | Create a new customer profile |
| `GET` | `/api/v1/customers` | Search customers by CPF, email, or phone (paginated) |
| `GET` | `/api/v1/customers/{id}` | Get a customer by internal ID |
| `PUT` | `/api/v1/customers/{id}` | Update customer profile fields |

All endpoints require a valid service-to-service JWT token. Responses follow the standard CredityFlow envelope format with `data`, `meta`, and `errors` fields.

### Example: Create Customer

```
POST /api/v1/customers
Content-Type: application/json

{
  "cpf": "123.456.789-00",
  "full_name": "Maria da Silva",
  "email": "maria@example.com",
  "phone": "+5511999990000",
  "address": {
    "street": "Rua das Flores",
    "number": "123",
    "neighborhood": "Centro",
    "city": "Sao Paulo",
    "state": "SP",
    "zip_code": "01001-000"
  }
}
```

## Configuration

Key environment variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | `jdbc:postgresql://localhost:5432/customer_db` |
| `AWS_SNS_TOPIC_ARN` | SNS topic for customer events | - |
| `AWS_SQS_LEAD_QUEUE_URL` | SQS queue for lead_qualified events | - |
| `KMS_KEY_ID` | AWS KMS key for PII encryption | - |
| `SERVER_PORT` | HTTP server port | `8080` |

Spring profiles: `local`, `staging`, `production`. The `local` profile disables KMS encryption and uses plaintext storage.

## Deployment

Deployed to AWS EKS via Helm chart. The CI/CD pipeline runs on GitHub Actions:

1. Build and test
2. Docker image push to ECR
3. Helm upgrade to the target EKS cluster

Resource requirements:
- **CPU**: 500m request / 1000m limit
- **Memory**: 512Mi request / 1024Mi limit
- **Replicas**: 2 (staging), 3 (production)

The service requires a PostgreSQL RDS instance provisioned via Terraform. Database migrations run automatically on startup using Flyway.

## Monitoring

**Metrics** (exposed via Micrometer to Prometheus):
- `customer_creation_total` -- counter of new customer records
- `customer_lookup_duration_seconds` -- histogram of GET request latency
- `duplicate_cpf_rejection_total` -- counter of duplicate CPF attempts

**Alerts**:
- POST error rate > 5% over a 5-minute window
- Database connection pool utilization > 90%
- Dead-letter queue depth > 0 for the `lead_qualified` consumer

**Dashboard**: "Customer Service Overview" in Grafana covers creation rate, lookup latency, and error breakdowns.

## Known Issues

- **LGPD erasure**: Data deletion requests are currently handled through a manual runbook. An automated endpoint is planned but not yet implemented.
- **Read replica lag**: Under heavy load, read queries may return stale data for up to 2 seconds when hitting the read replica.
- **No soft delete**: Customer records are hard-deleted, which complicates audit trails for deleted profiles.
