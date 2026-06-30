# Credit Policy Service

| | |
|---|---|
| **Service ID** | svc-21 |
| **Domain** | Credit Analysis |
| **Owner** | team-credit |
| **Stack** | Kotlin / Spring Boot |

## Overview

Every loan has rules. Maximum loan-to-value ratios. Minimum credit scores. Debt-to-income ceilings. The Credit Policy Service is where those rules live, and where every proposal goes to be judged against them.

The service maintains a **versioned rule engine** — policy rules can be added, modified, and retired without code deployments. Each evaluation is recorded with the exact policy version used, so any decision can be reproduced months or years later for audit purposes. The risk management team manages rules through a backoffice UI that talks to this service's REST API.

When a credit score arrives (via the `credit_scored` event), the service automatically evaluates all applicable rules and publishes the result. The downstream `credit-decision-service` uses this evaluation to render the final approve/deny/review outcome.

## Getting Started

```bash
# Local dependencies
docker-compose up -d postgres kafka

# Environment variables
export DATABASE_URL=jdbc:postgresql://localhost:5432/credit_policy_db
export DATABASE_USERNAME=policy_svc
export DATABASE_PASSWORD=changeme
export KAFKA_BOOTSTRAP_SERVERS=localhost:9092

# Build and run
./gradlew bootRun

# Run tests
./gradlew test
```

**Database migrations** run on startup via Flyway. Seed data for default policy rules is included in `V2__seed_default_policies.sql`.

## Architecture

The rule engine is the heart of this service. Rules are stored as structured records in PostgreSQL — not as code — with fields for the target metric (e.g., `LTV`, `DTI`, `CREDIT_SCORE`), comparison operator, threshold value, and applicability criteria (product type, collateral type).

Evaluation proceeds as follows:
1. Load the active `PolicyVersion` (or a specific version for simulation)
2. Filter rules applicable to the proposal's product and collateral type
3. Evaluate each rule against the proposal's financial metrics
4. Aggregate results: if any critical rule fails, the overall outcome is FAIL; if only advisory rules fail, outcome is CONDITIONAL

This service does **not** make the final credit decision — that responsibility belongs to `credit-decision-service`. Separation of concerns keeps the policy logic pure and testable.

Detailed diagrams and ADRs are in [architecture.md](./architecture.md).

## API Reference

### GET /api/v1/policies

Returns active policy rules. Supports query parameters:
- `product_type` — filter by loan product (e.g., `VEHICLE_LOAN`, `HOME_EQUITY`)
- `effective_date` — return rules effective on a specific date

### POST /api/v1/policies/evaluate

Evaluates rules against a proposal payload.

**Request:**
```json
{
  "proposal_id": "uuid",
  "product_type": "VEHICLE_LOAN",
  "collateral_type": "VEHICLE",
  "credit_score": 720,
  "ltv_ratio": 0.65,
  "dti_ratio": 0.35,
  "loan_amount": 50000.00,
  "customer_age": 34
}
```

**Response:**
```json
{
  "evaluation_id": "uuid",
  "policy_version": "v14",
  "aggregate_outcome": "PASS",
  "rule_results": [
    {"rule_id": "r-01", "name": "Max LTV Vehicle", "result": "PASS", "threshold": 0.70, "actual": 0.65},
    {"rule_id": "r-02", "name": "Min Credit Score", "result": "PASS", "threshold": 600, "actual": 720},
    {"rule_id": "r-03", "name": "Max DTI", "result": "PASS", "threshold": 0.40, "actual": 0.35}
  ]
}
```

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `DATABASE_URL` | JDBC connection string | — |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka brokers | `localhost:9092` |
| `KAFKA_CONSUMER_GROUP` | Consumer group | `credit-policy-cg` |
| `POLICY_CACHE_ENABLED` | Cache active policy version in memory | `true` |
| `POLICY_CACHE_TTL_SECONDS` | In-memory cache TTL | `60` |

## Deployment

Helm chart deployment to EKS:

```bash
helm upgrade --install credit-policy-service ./charts/credit-policy-service \
  --namespace credit-analysis \
  --values values-production.yaml
```

Minimal resource footprint: 256Mi memory request, 128m CPU. The service is stateless (database-backed) and scales horizontally with ease. HPA target: 2-4 replicas, 65% CPU utilisation.

## Monitoring

- **Dashboard:** "Policy Evaluation Overview" — volume, pass/fail ratio, latency distribution
- **Dashboard:** "Rule Effectiveness" — per-rule failure rates and trends

**Alerts:**
- `policy_evaluation_error_rate` — errors > 2% over 5 min
- `policy_all_fail_spike` — FAIL rate jumps 20% within 1 hour (may indicate misconfigured rules)

## Known Issues

1. **No rule conflict detection** — The system does not automatically detect when two rules produce contradictory guidance (e.g., one rule approves a range that another rule denies). The risk team must review rule interactions manually. A rule conflict analyser is on the roadmap (CRED-2301).

2. **In-memory cache staleness** — When `POLICY_CACHE_ENABLED=true`, rule updates may not take effect across all pods for up to `POLICY_CACHE_TTL_SECONDS`. During this window, different pods may evaluate against different policy versions. Set TTL to 0 for immediate consistency at the cost of higher database load.

3. **No dry-run for batch evaluation** — The simulation endpoint evaluates one proposal at a time. The risk team has requested batch simulation (e.g., "how would this rule change affect the last 10,000 proposals?"). Planned for Q3 2026.
