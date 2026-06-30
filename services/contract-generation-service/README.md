# Contract Generation Service

## Overview

The contract-generation-service is the workhorse behind CredityFlow's contract creation pipeline. When a collateral appraisal wraps up, this service kicks in — it pulls the approved loan terms from the proposal, picks the right template, and produces a fully rendered contract PDF. The output is a draft contract ready for compliance review and, eventually, digital signature.

This service sits at the boundary between the Analysis domain and the Contract domain. It's the first service in the Contract subdomain pipeline: **contract generation → compliance check → digital signature → contract storage**.

**Owner:** team-contracts
**Language:** Kotlin / Spring Boot
**Database:** PostgreSQL (`contract_db`)

## Getting Started

### Prerequisites

- JDK 17+
- Docker and Docker Compose (for local PostgreSQL)
- Access to the internal Nexus repository for shared libraries

### Running Locally

```bash
# Start dependencies
docker-compose up -d postgres

# Run the service
./gradlew bootRun --args='--spring.profiles.active=local'
```

The service will start on port `8080` by default. Swagger UI is available at `http://localhost:8080/swagger-ui.html`.

### Running Tests

```bash
./gradlew test          # Unit tests
./gradlew integrationTest  # Integration tests (requires Docker)
```

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `DB_HOST` | PostgreSQL host | `localhost` |
| `DB_PORT` | PostgreSQL port | `5432` |
| `DB_NAME` | Database name | `contract_db` |
| `KAFKA_BROKERS` | Kafka bootstrap servers | `localhost:9092` |
| `PROPOSAL_SERVICE_URL` | Base URL for proposal-service | `http://proposal-service:8080` |
| `TEMPLATE_STORAGE_PATH` | Path to contract template files | `/opt/templates` |

## Architecture

The service follows a fairly standard hexagonal architecture pattern. Inbound adapters handle Kafka events and REST requests; outbound adapters talk to PostgreSQL and the proposal-service.

### Key Components

- **ContractGenerationUseCase** — core business logic for template selection and rendering.
- **TemplateEngine** — wraps the Thymeleaf-based PDF renderer. Templates are stored as HTML with token placeholders.
- **ProposalClient** — Feign client that fetches proposal and loan terms from proposal-service.
- **ContractRepository** — JPA repository for contract metadata persistence.

### Template System

Contract templates are versioned HTML files stored in the database (the `contract_template` table). The legal team uploads new versions through a backoffice tool. Each template has conditional clauses that are included or excluded based on the loan type and collateral type.

**Tribal knowledge:** The template rendering uses Thymeleaf in "standalone" mode, not the typical Spring MVC integration. This was done because the PDF renderer (OpenHTMLToPDF) needs raw HTML, not a Spring view. If you're debugging template issues, look at `TemplateRenderService`, not the Spring view resolver.

### Data Flow

1. `appraisal_completed` event arrives on Kafka
2. Service extracts the `proposalId` from the event payload
3. REST call to proposal-service to fetch `LoanTerms`
4. Template selected based on `loanType` + `collateralType`
5. HTML rendered via Thymeleaf, converted to PDF via OpenHTMLToPDF
6. PDF stored (reference saved in `contract_db`)
7. `contract_generated` event published

## API Reference

### POST /api/v1/contracts/generate

Triggers contract generation. Typically called automatically via Kafka, but this endpoint exists for manual re-generation or backoffice use.

**Request Body:**
```json
{
  "proposalId": "uuid",
  "templateOverride": "VEHICLE_STANDARD_V3"  // optional
}
```

**Response:** `201 Created`
```json
{
  "contractId": "uuid",
  "status": "GENERATED",
  "pdfUrl": "/api/v1/contracts/{id}/pdf"
}
```

### GET /api/v1/contracts/{id}

Returns contract metadata and status.

## Configuration

### Template Management

Templates follow the naming convention `{LOAN_TYPE}_{COLLATERAL_TYPE}_V{version}`. The active template for each combination is marked in the `contract_template` table with `is_active = true`. Only one template per combination can be active at a time.

When the legal team needs to update a template, the process is:
1. Upload new template version via backoffice
2. Run compliance validation on the new template (manual step)
3. Mark new version as active, which deactivates the previous one

**Important:** Never delete old template versions. Contracts reference the template version they were generated with, and auditors need to see exactly which template was used.

### Kafka Topics

| Topic | Direction | Consumer Group |
|-------|-----------|----------------|
| `credityflow.appraisal.completed` | Consume | `contract-generation-cg` |
| `credityflow.contract.generation.requested` | Produce | — |
| `credityflow.contract.generated` | Produce | — |

## Deployment

Deployed as a Kubernetes Deployment in the `contract` namespace. Typically 2 replicas in production.

**Resource requirements:**
- Memory: 512Mi request / 1Gi limit (PDF generation is memory-intensive)
- CPU: 250m request / 500m limit

The service has a liveness probe at `/actuator/health/liveness` and a readiness probe at `/actuator/health/readiness`.

## Monitoring

### Key Metrics

- `contract_generation_duration_seconds` — track p50/p95/p99; PDF rendering is the bottleneck
- `contracts_generated_total` — broken down by template type
- `template_render_errors_total` — any non-zero value should be investigated

### Alerts

- **High failure rate** — triggers if more than 2% of generation attempts fail within 15 minutes
- **Slow generation** — triggers if p99 exceeds 30 seconds

### Dashboard

The "Contract Generation Overview" dashboard in Grafana shows real-time generation volume, latency distribution, and error rates.

## Known Issues

1. **Memory spikes during concurrent PDF generation.** OpenHTMLToPDF holds the entire document in memory. Under load tests with 50 concurrent generations, we've seen OOM kills. The current mitigation is a semaphore that limits concurrent PDF renders to 10. If you need to increase throughput, scale horizontally rather than raising the semaphore limit.

2. **Template caching.** Templates are cached in-memory for 5 minutes. If the legal team activates a new template, it can take up to 5 minutes for the service to pick it up. There's no cache invalidation endpoint yet — it's on the backlog.

3. **Proposal-service dependency.** If proposal-service is down, contract generation fails. There's a retry with 3 attempts and exponential backoff, but no circuit breaker yet. The team has discussed adding Resilience4j but hasn't prioritized it.
