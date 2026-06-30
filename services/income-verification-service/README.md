# Income Verification Service

> **svc-19** | team-credit | Credit Analysis domain

---

## Overview

Not everyone is truthful about how much they earn. The Income Verification Service exists to catch the gap between what a borrower *says* they make and what the evidence actually shows.

It cross-references declared income against three sources: credit bureau employment data (from `credit-bureau-integration`), uploaded payslips and tax returns (stored by `customer-service`), and historical income patterns. The output is a `VerificationResult` that tells downstream services whether the declared income is credible and by how much it might differ from reality.

Built with Kotlin and Spring Boot, the service stores all verification data in a dedicated PostgreSQL database (`income_db`) to maintain a complete audit trail — a regulatory requirement for lending operations in Brazil.

## Getting Started

**Tech stack:** Kotlin 1.9, Spring Boot 3.x, PostgreSQL 15, Apache Kafka

```bash
# Prerequisites
docker-compose up -d postgres kafka    # or use your local instances

# Environment
export DATABASE_URL=jdbc:postgresql://localhost:5432/income_db
export DATABASE_USERNAME=income_svc
export DATABASE_PASSWORD=<password>
export KAFKA_BOOTSTRAP_SERVERS=localhost:9092

# Run
./gradlew bootRun

# Test
./gradlew test
./gradlew integrationTest
```

The application binds to port `8080` by default. Health check: `GET /actuator/health`.

Database migrations run automatically on startup via Flyway.

## Architecture

The service sits between the bureau data retrieval step and the credit scoring step in the pipeline:

```
bureau_data_fetched --> [income-verification-service] --> income_verified
```

Internally, it uses a layered architecture:
- **API layer** — Spring MVC controllers for REST endpoints
- **Application layer** — orchestrates verification logic, coordinates data sources
- **Domain layer** — `IncomeRecord`, `VerificationResult`, `IncomeSource` entities and business rules
- **Infrastructure layer** — Kafka consumers/producers, JPA repositories, HTTP clients

The verification algorithm computes a confidence score (0.0 to 1.0) based on how closely the declared income matches each evidence source. Thresholds for VERIFIED (>= 0.8), PARTIALLY_VERIFIED (0.5-0.8), and UNVERIFIED (< 0.5) are configurable.

See [architecture.md](./architecture.md) for component diagrams and ADRs.

## API Reference

### POST /api/v1/income/verify

Triggers income verification for a customer/proposal.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `proposal_id` | UUID | Yes | The loan proposal identifier |
| `customer_id` | UUID | Yes | The customer identifier |
| `declared_income` | Decimal | Yes | Monthly gross income declared by customer |
| `income_currency` | String | No | ISO 4217 code (default: BRL) |

**Response (200):**
```json
{
  "verification_id": "uuid",
  "status": "VERIFIED",
  "confidence_score": 0.92,
  "verified_income": 8500.00,
  "discrepancy_pct": 5.5,
  "sources_checked": ["BUREAU", "PAYSLIP"],
  "verified_at": "2026-03-29T14:30:00Z"
}
```

### GET /api/v1/income/{customer_id}

Returns the latest income records and verification results for a customer. Supports `?proposal_id=` query parameter to filter by proposal.

## Configuration

Key configuration properties (set via environment variables or `application.yaml`):

| Property | Description | Default |
|----------|-------------|---------|
| `VERIFICATION_CONFIDENCE_VERIFIED` | Minimum confidence for VERIFIED status | `0.8` |
| `VERIFICATION_CONFIDENCE_PARTIAL` | Minimum confidence for PARTIALLY_VERIFIED | `0.5` |
| `CUSTOMER_SERVICE_URL` | Base URL for customer-service | — |
| `DATABASE_URL` | JDBC connection string for income_db | — |
| `KAFKA_CONSUMER_GROUP` | Kafka consumer group ID | `income-verification-cg` |

## Deployment

Runs on AWS EKS in the `credit-analysis` namespace. Deployed via Helm.

```bash
helm upgrade --install income-verification ./charts/income-verification-service \
  --namespace credit-analysis \
  --values values-production.yaml
```

**Scaling:** Horizontal pod autoscaler configured for 2-6 replicas based on CPU (target 70%). Kafka partition count should match max replica count for optimal parallelism.

**Database:** AWS RDS PostgreSQL (db.r6g.large) with read replicas for query endpoints.

## Monitoring

**Dashboards:**
- *Income Verification Overview* — request volume, status breakdown, processing latency
- *Discrepancy Analysis* — trends in income discrepancies by source type

**Key alerts:**
- `verification_failure_rate_high` — failure rate > 10% over 10 min
- `verification_latency_p95_high` — p95 response time > 5s

All log entries include `trace_id`, `proposal_id`, and `customer_id` for end-to-end correlation.

## Known Issues

1. **Bureau employment data lag** — Employment records from ServicosBureau can be 30-60 days behind. Customers who recently changed jobs may show discrepancies even when their declared income is accurate. The confidence algorithm weights bureau employment data lower for short-tenure records, but this is imperfect.

2. **Document OCR limitations** — Payslip and tax return validation relies on metadata tags set by the document service. If OCR quality is poor or the document format is non-standard, income extraction may fail silently, resulting in lower confidence scores rather than explicit errors.

3. **Concurrent verification requests** — If `bureau_data_fetched` is published multiple times for the same proposal (e.g., due to a retry), duplicate verifications can occur. An idempotency check on `(proposal_id, customer_id)` prevents duplicate persistence, but duplicate Kafka events may still be published. Fix tracked in CRED-2103.
