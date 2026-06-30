# billing-service

**Service ID:** svc-38 | **Domain:** Collections | **Owner:** team-collections | **Status:** Production

The billing-service is the financial backbone of CredityFlow's collections pipeline. It translates approved loan disbursements into concrete installment schedules, reconciles incoming payments, and raises the alarm when borrowers miss due dates. Every downstream collection process -- dunning, negotiation, legal -- depends on the accuracy of data produced here.

## Getting Started

### Prerequisites

- JDK 17+
- Docker & Docker Compose (for local PostgreSQL and Kafka)
- Gradle 8.x

### Running Locally

```bash
# Start infrastructure
docker-compose up -d postgres kafka

# Run the service
./gradlew bootRun --args='--spring.profiles.active=local'
```

The service starts on port `8080` by default. Verify with:

```bash
curl http://localhost:8080/actuator/health
```

### Environment Variables

| Variable | Description | Default |
|---|---|---|
| `SPRING_DATASOURCE_URL` | JDBC URL for billing_db | `jdbc:postgresql://localhost:5432/billing_db` |
| `SPRING_KAFKA_BOOTSTRAP_SERVERS` | Kafka broker addresses | `localhost:9092` |
| `BILLING_OVERDUE_CRON` | Cron expression for the overdue detection job | `0 0 6 * * *` |

## Architecture

billing-service follows a standard Spring Boot layered architecture with event-driven integration. It consumes `disbursement_done` and `negotiation_accepted` events from Kafka, processes business logic through a service layer, persists data to PostgreSQL (`billing_db`), and publishes `installment_created`, `installment_due`, and `overdue_installment` events back to Kafka.

Core entities are **Installment**, **PaymentRecord**, and **InstallmentSchedule**. The amortization engine supports multiple calculation strategies (Price, SAC) selected per product type.

For a detailed component diagram and ADRs, see [architecture.md](./architecture.md).

## API Reference

### GET /api/v1/billing/{proposal_id}/installments

Returns the full installment schedule for a proposal.

**Response 200:**
```json
{
  "proposal_id": "prop-123",
  "installments": [
    {
      "number": 1,
      "amount": 1500.00,
      "due_date": "2026-04-15",
      "status": "PAID"
    }
  ]
}
```

### GET /api/v1/billing/overdue

Returns paginated list of overdue installments. Supports query parameters `page`, `size`, and `min_days_overdue`.

**Response 200:**
```json
{
  "content": [
    {
      "installment_id": "inst-456",
      "proposal_id": "prop-789",
      "amount": 1200.00,
      "due_date": "2026-03-01",
      "days_overdue": 28
    }
  ],
  "page": 0,
  "total_pages": 5
}
```

## Configuration

Application configuration lives in `src/main/resources/application.yml` with profile-specific overrides. Key tuning knobs:

- **`billing.amortization.default-method`** -- `PRICE` or `SAC`. Determines the default installment calculation method.
- **`billing.overdue.grace-period-days`** -- Number of days after the due date before an installment is considered overdue (default: 0).
- **`billing.kafka.consumer.concurrency`** -- Number of concurrent Kafka consumer threads (default: 3).

Secrets are injected via Kubernetes secrets and never stored in config files.

## Deployment

The service is packaged as a Docker image and deployed to Kubernetes via Helm charts stored in the `deploy/` directory.

```bash
# Build image
./gradlew bootBuildImage --imageName=credityflow/billing-service:latest

# Deploy to staging
helm upgrade --install billing-service deploy/helm \
  --namespace collections \
  --values deploy/helm/values-staging.yaml
```

Database migrations run automatically on startup via Flyway. New migrations go in `src/main/resources/db/migration/`.

## Monitoring

| Metric | Description |
|---|---|
| `billing_installments_created_total` | Total installment schedules generated |
| `billing_overdue_detected_total` | Overdue installments flagged |
| `billing_payment_matched_total` | Payments successfully reconciled |
| `billing_schedule_generation_duration_seconds` | Schedule generation latency |

Health check is available at `/actuator/health`. Prometheus metrics are exposed at `/actuator/prometheus`.

**Alerts:**
- *HighOverdueRate* -- fires when overdue detections exceed 500/hour.
- *ScheduleGenerationFailures* -- fires on more than 10 generation errors in 5 minutes.

Dashboards are available in Grafana under the "Collections > Billing" folder.

## Known Issues

1. **Bulk disbursement spikes:** During month-end bulk disbursements, Kafka consumer lag can increase significantly. The team is evaluating partition scaling to address this.
2. **Timezone handling:** Overdue detection uses UTC. Borrowers in edge-case timezones may see overdue flags slightly before their local due date passes.
3. **Negotiation replay:** If a `negotiation_accepted` event is delivered more than once, the idempotency check relies on offer ID deduplication. Ensure Kafka consumer `enable.auto.commit` is `false` for at-least-once semantics.
