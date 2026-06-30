# proposal-service

## Overview

The proposal-service sits at the heart of CredityFlow's lending workflow. It orchestrates the entire loan proposal lifecycle, from the moment a customer submits a loan request through credit analysis, fraud checks, document collection, appraisal, and finally disbursement. Think of it as the "conductor" of the lending orchestra -- it does not perform the individual checks itself, but it tracks every stage, enforces ordering, and decides when a proposal is ready to move forward.

Proposals can be for two product types: **vehicle-backed loans** and **home-backed loans**. Each carries different collateral requirements, document checklists, and approval criteria, all managed through configurable product rules.

Owned by **team-onboarding**. Domain: **Onboarding**.

## Getting Started

**Prerequisites**: JDK 17+, Docker Compose, AWS CLI.

```bash
docker-compose up -d postgres
./gradlew bootRun --args='--spring.profiles.active=local'
```

The local profile stubs out calls to customer-service and eligibility-service so you can develop in isolation. To run with real service dependencies, use the `staging` profile with the VPN connected.

**Tests**:
```bash
./gradlew test                    # Unit + domain tests
./gradlew integrationTest         # Requires Docker for Testcontainers
./gradlew contractTest            # Pact contract tests against customer-service
```

## Architecture

Kotlin/Spring Boot application with a state-machine-driven domain model. The proposal moves through states (`DRAFT -> SUBMITTED -> ANALYZING -> APPROVED -> DISBURSED` or `REJECTED` at various points) based on events received from downstream services.

Synchronous dependencies:
- **customer-service** (validate customer exists)
- **eligibility-service** (check basic eligibility)

Asynchronous dependencies (via SNS/SQS):
- fraud-analysis-service, credit-engine, appraisal-service, disbursement-service

See [architecture.md](architecture.md) for detailed diagrams and ADR notes.

## API Reference

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/proposals` | Create a new loan proposal |
| `GET` | `/api/v1/proposals` | List proposals (filterable by status, customer, date range) |
| `GET` | `/api/v1/proposals/{id}` | Get full proposal details |
| `PUT` | `/api/v1/proposals/{id}` | Update proposal (only in editable states) |

### Creating a Proposal

```
POST /api/v1/proposals
{
  "customer_id": "cust-abc-123",
  "product_type": "VEHICLE_LOAN",
  "requested_amount": 45000.00,
  "term_months": 48,
  "collateral": {
    "type": "VEHICLE",
    "make": "Toyota",
    "model": "Corolla",
    "year": 2022,
    "plate": "ABC1D23",
    "fipe_code": "015267-0"
  }
}
```

The service validates the customer via customer-service, checks eligibility, persists the proposal in `DRAFT` status, and publishes a `proposal_started` event.

## Configuration

| Variable | Description |
|----------|-------------|
| `DATABASE_URL` | PostgreSQL connection string for proposal_db |
| `CUSTOMER_SERVICE_URL` | Base URL for customer-service REST calls |
| `ELIGIBILITY_SERVICE_URL` | Base URL for eligibility-service REST calls |
| `AWS_SNS_PROPOSAL_TOPIC_ARN` | SNS topic for proposal events |
| `AWS_SQS_*_QUEUE_URL` | SQS queues for each consumed event type |

Circuit breakers (Resilience4j) are configured for both synchronous dependencies with a 5-second timeout, 50% failure threshold, and 30-second half-open window.

## Deployment

Runs on AWS EKS. Deployment via Helm chart through GitHub Actions pipeline.

- **Replicas**: 2 (staging), 4 (production) -- higher count due to event processing load
- **CPU**: 750m / 1500m
- **Memory**: 768Mi / 1536Mi
- **HPA**: scales on CPU (70% threshold) and SQS queue depth

Database migrations managed by Flyway. The proposal_db schema includes partitioning by `created_at` month for the status history table.

## Monitoring

Dashboards:
- **Proposal Pipeline**: funnel visualization from creation to disbursement, with conversion rates between stages
- **Proposal Latency Breakdown**: time spent in each state, helping identify bottlenecks
- **Event Processing Health**: SQS lag, DLQ depth, message throughput

Key alerts:
- Proposals stuck in `ANALYZING` for more than 48 hours
- Rejection rate above 60% in a 24-hour rolling window
- Any dead-letter queue depth above zero
- Synchronous dependency failure rate above 5%

## Known Issues

- **Co-borrower support**: Not yet implemented. Proposals currently support a single applicant only. This is a known gap for home-backed loans where couples often apply together.
- **State machine recovery**: If a service is down for an extended period, proposals can get stuck. There is a manual "nudge" admin endpoint (`POST /admin/proposals/{id}/retry`) but no automated retry mechanism.
- **Product rule configuration**: Loan term limits and collateral rules are currently hardcoded. Migration to a configuration service is planned for Q3.
