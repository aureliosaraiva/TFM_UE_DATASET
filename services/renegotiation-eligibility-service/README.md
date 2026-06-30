# renegotiation-eligibility-service

`svc-43` -- Collections Domain -- team-collections -- Production

## Overview

Not every borrower should be allowed to renegotiate their debt. The renegotiation-eligibility-service acts as a gatekeeper, evaluating whether a borrower qualifies based on a configurable set of business rules. It checks things like how many days overdue the debt is, whether the borrower has already renegotiated before, and what risk tier they fall into.

When a borrower passes all rules, the service publishes a `renegotiation_eligible` event that unlocks the simulation pipeline. When they fail, the service returns a clear explanation of which rules were not met, useful both for internal operations and for surfacing feedback in self-service channels.

## Getting Started

**Stack:** Kotlin, Spring Boot, PostgreSQL, Kafka, Gradle

```bash
# Start dependencies
docker-compose up -d postgres kafka

# Run the service
./gradlew bootRun --args='--spring.profiles.active=local'
```

Default port: `8084`. Health: `http://localhost:8084/actuator/health`.

### Environment Variables

| Variable | Description | Default |
|---|---|---|
| `SPRING_DATASOURCE_URL` | JDBC URL for renegotiation_db | `jdbc:postgresql://localhost:5432/renegotiation_db` |
| `SPRING_KAFKA_BOOTSTRAP_SERVERS` | Kafka broker addresses | `localhost:9092` |
| `ELIGIBILITY_BILLING_SERVICE_URL` | billing-service base URL | `http://localhost:8080` |
| `ELIGIBILITY_CUSTOMER_SERVICE_URL` | customer-service base URL | `http://localhost:8090` |

## Architecture

The service has a clean request-response flow layered on top of event publishing. When a check-eligibility request comes in, it synchronously calls billing-service and customer-service to gather the data it needs, evaluates every active rule in sequence, and returns the result. If eligible, an event goes out on Kafka.

The rule engine is deliberately simple: rules are rows in a database table, each with a type, parameters, and an active flag. There is no complex rule DSL. This keeps the system easy to audit and debug, which matters when operations staff need to understand why a specific borrower was denied.

Entities: **RenegotiationEligibility**, **EligibilityRule**, **EligibilityResult**.

Database: `renegotiation_db` (shared with renegotiation-simulation-service).

Detailed diagrams and ADRs are in [architecture.md](./architecture.md).

## API Reference

### POST /api/v1/renegotiation/check-eligibility

Check if a borrower is eligible for renegotiation.

**Request:**
```json
{
  "borrower_id": "bor-100",
  "proposal_id": "prop-200"
}
```

**Response 200 (eligible):**
```json
{
  "eligible": true,
  "borrower_id": "bor-100",
  "proposal_id": "prop-200",
  "rules_evaluated": 5,
  "rules_passed": 5,
  "checked_at": "2026-03-29T11:00:00Z"
}
```

**Response 200 (ineligible):**
```json
{
  "eligible": false,
  "borrower_id": "bor-100",
  "proposal_id": "prop-200",
  "rules_evaluated": 5,
  "rules_passed": 3,
  "failed_rules": [
    { "rule": "MAX_RENEGOTIATIONS", "reason": "Borrower has already renegotiated 2 times (max: 2)" },
    { "rule": "MIN_DAYS_OVERDUE", "reason": "Debt is 15 days overdue (min: 30)" }
  ],
  "checked_at": "2026-03-29T11:00:00Z"
}
```

### GET /api/v1/renegotiation/rules

Returns all active eligibility rules. Useful for back-office tooling and rule auditing.

```json
{
  "rules": [
    { "id": "rule-1", "type": "MIN_DAYS_OVERDUE", "params": { "min_days": 30 }, "active": true },
    { "id": "rule-2", "type": "MAX_RENEGOTIATIONS", "params": { "max_count": 2 }, "active": true }
  ]
}
```

## Configuration

Rules are managed in the database, not in config files. This allows operations teams to adjust eligibility criteria without redeploying the service. The `GET /api/v1/renegotiation/rules` endpoint provides read access; modifications go through an internal admin API with approval workflow.

Application-level settings:

- **`eligibility.cache.ttl-minutes`** -- How long to cache billing/customer data per check (default: 5).
- **`eligibility.rule-refresh-interval-seconds`** -- How often rules are reloaded from the database (default: 60).

## Deployment

```bash
./gradlew bootBuildImage --imageName=credityflow/renegotiation-eligibility-service:latest

helm upgrade --install reneg-eligibility deploy/helm \
  --namespace collections \
  --values deploy/helm/values-staging.yaml
```

Flyway migrations in `src/main/resources/db/migration/`. Since `renegotiation_db` is shared with the simulation service, coordinate migration ordering carefully.

## Monitoring

Key metrics at `/actuator/prometheus`:

- `renegotiation_eligibility_checks_total` -- total checks performed
- `renegotiation_eligible_total` -- checks that returned eligible
- `renegotiation_ineligible_total` -- checks that returned ineligible
- `renegotiation_rule_evaluation_duration_seconds` -- rule engine latency

**Alerts:**
- *HighIneligibilityRate* -- if more than 95% of checks fail, something may be wrong with the rules or upstream data.
- *RuleEvaluationSlow* -- P99 evaluation time over 2 seconds, typically caused by slow upstream service calls.

## Known Issues

- **Upstream dependency failures:** If billing-service or customer-service is down, eligibility checks will fail. There is no fallback mode; the service returns a `503` and the caller should retry.
- **Rule ordering:** Rules are evaluated in insertion order. There is no explicit priority field yet, which can cause confusion when rules interact. A priority field is planned.
- **Shared database:** Schema changes in `renegotiation_db` must be coordinated with renegotiation-simulation-service to avoid migration conflicts.
