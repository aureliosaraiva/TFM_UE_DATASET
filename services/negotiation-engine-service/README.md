# negotiation-engine-service

| | |
|---|---|
| **Service ID** | svc-41 |
| **Domain** | Collections |
| **Owner** | team-collections |
| **Status** | Production |

## Overview

The negotiation-engine-service generates settlement offers for overdue borrowers. It can propose discounts on outstanding balances, extend payment timelines, or reduce interest rates -- whatever combination the business rules allow. The goal is straightforward: recover as much debt as possible while giving borrowers a realistic path back to good standing.

This service sits between the billing-service (which tells it how much is owed) and the customer-service (which provides borrower risk profiles). When a borrower accepts an offer, the service publishes a `negotiation_accepted` event that triggers billing-service to recalculate the installment schedule.

## Getting Started

```bash
# Prerequisites: JDK 17+, Docker Compose, Gradle 8.x

# Spin up dependencies
docker-compose up -d postgres kafka

# Start the service
./gradlew bootRun --args='--spring.profiles.active=local'

# Verify
curl -s http://localhost:8082/actuator/health | jq .
```

### Environment Variables

| Variable | Description |
|---|---|
| `SPRING_DATASOURCE_URL` | JDBC URL for collections_db |
| `SPRING_KAFKA_BOOTSTRAP_SERVERS` | Kafka broker list |
| `NEGOTIATION_BILLING_SERVICE_URL` | Base URL of billing-service |
| `NEGOTIATION_CUSTOMER_SERVICE_URL` | Base URL of customer-service |
| `NEGOTIATION_MAX_DISCOUNT_PCT` | Global cap on discount percentage (default: 40) |

## Architecture

The service follows a clean separation between simulation (stateless, read-only) and acceptance (stateful, event-producing). The simulation endpoint calls billing-service and customer-service via synchronous REST, runs discount rules through a lightweight rule engine, and returns results without persisting anything. The acceptance endpoint transitions a persisted offer to ACCEPTED and publishes the `negotiation_accepted` event.

**Entities:**
- `NegotiationOffer` -- the concrete offer with expiration and status lifecycle
- `NegotiationTerms` -- financial details (discount %, new dates, revised rate)
- `DiscountRule` -- configurable rules governing what discounts are permissible

Data is stored in `collections_db` (PostgreSQL) and contains financial data. See [architecture.md](./architecture.md) for the full component diagram.

## API Reference

### POST /api/v1/negotiations/simulate

Simulate negotiation scenarios without persisting. Requires `borrower_id` and `proposal_id` in the request body.

**Request:**
```json
{
  "borrower_id": "bor-100",
  "proposal_id": "prop-200"
}
```

**Response 200:**
```json
{
  "scenarios": [
    {
      "type": "DISCOUNT",
      "discount_pct": 15.0,
      "new_total": 8500.00,
      "valid_until": "2026-04-15T23:59:59Z"
    },
    {
      "type": "EXTENSION",
      "additional_months": 6,
      "new_monthly_amount": 650.00,
      "valid_until": "2026-04-15T23:59:59Z"
    }
  ]
}
```

### POST /api/v1/negotiations/{id}/accept

Accept a previously created offer. Returns `409` if the offer has expired or was already accepted/rejected.

**Response 200:**
```json
{
  "offer_id": "offer-555",
  "status": "ACCEPTED",
  "accepted_at": "2026-03-29T10:00:00Z"
}
```

## Configuration

Discount rules are stored in the `discount_rules` database table and can be managed through an internal admin API (not exposed externally). Each rule specifies:

- Applicability criteria (days overdue range, risk tier, product type)
- Discount bounds (min/max percentage)
- Priority and active/inactive flag

The global discount cap (`NEGOTIATION_MAX_DISCOUNT_PCT`) acts as a safety net above all rules.

## Deployment

Standard Helm-based deployment to Kubernetes:

```bash
./gradlew bootBuildImage --imageName=credityflow/negotiation-engine-service:latest

helm upgrade --install negotiation-engine deploy/helm \
  --namespace collections \
  --values deploy/helm/values-staging.yaml
```

Database: Flyway migrations in `src/main/resources/db/migration/`. The service shares `collections_db` -- coordinate migration ordering with other services using the same database.

## Monitoring

| Metric | What it tells you |
|---|---|
| `negotiation_offers_created_total` | Volume of offers generated |
| `negotiation_offers_accepted_total` | Successful acceptances |
| `negotiation_simulation_duration_seconds` | How long simulations take |
| `negotiation_acceptance_rate` | Ratio of accepted vs. total offers |

Alerts fire when the acceptance rate drops below 10% (may indicate offers are not competitive) or when simulation P99 latency exceeds 3 seconds (usually means billing-service or customer-service is slow).

## Known Issues

1. **Stale billing data:** Simulations fetch billing data synchronously. If a payment arrives between simulation and acceptance, the offer may reference an outdated balance. Optimistic locking on the offer catches this, but the borrower must re-simulate.
2. **Rule engine performance:** With a large number of active discount rules (100+), rule evaluation adds measurable latency. Consider pruning inactive rules regularly.
3. **Shared database:** This service shares `collections_db` with other services. Schema migrations must be coordinated to avoid downtime.
