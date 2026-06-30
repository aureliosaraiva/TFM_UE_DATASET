# Credit Decision Service

**svc-22** // Credit Analysis // team-credit

## Overview

This is where the rubber meets the road. The Credit Decision Service takes the output of policy evaluation and renders a final, binding credit decision for each loan proposal: **APPROVED**, **DENIED**, or **MANUAL_REVIEW**.

Automated decisions flow through in milliseconds. Edge cases — proposals that narrowly fail a policy rule, high-value loans, or flagged accounts — are routed to a manual review queue where trained analysts can examine the details and issue an override if warranted. Every decision and every override is immutably recorded for regulatory compliance.

The service publishes specific outcome events (`credit_approved`, `credit_denied`) that trigger very different downstream workflows: approvals kick off collateral appraisal and contract generation, while denials trigger customer notification flows.

## Getting Started

### Prerequisites

- JDK 17+, Kotlin 1.9+
- PostgreSQL 15+
- Kafka broker

### Quick Start

```bash
# Spin up infrastructure
docker-compose up -d postgres kafka

# Configure
export DATABASE_URL=jdbc:postgresql://localhost:5432/credit_decisions_db
export DATABASE_USERNAME=decisions_svc
export DATABASE_PASSWORD=changeme
export KAFKA_BOOTSTRAP_SERVERS=localhost:9092

# Run
./gradlew bootRun

# Health check
curl http://localhost:8080/actuator/health
```

### Tests

```bash
./gradlew test                          # Unit tests
./gradlew integrationTest               # Integration (requires infra)
./gradlew test --tests '*DecisionLogicTest'  # Specific test class
```

## Architecture

The decision logic is intentionally simple by design. Complex risk evaluation belongs in `credit-policy-service` and `credit-scoring-engine`. This service's job is to:

1. Map policy outcomes to credit decisions using straightforward rules
2. Handle the manual review workflow (queue management, assignment, override)
3. Maintain the authoritative decision record

The override mechanism enforces an **approval chain**: junior analysts can override denials to manual review, but only senior analysts can override to approval. All overrides require a reason code from a controlled vocabulary.

```
credit_policy_evaluated --> [credit-decision-service] --> credit_approved
                                                     --> credit_denied
                                                     --> credit_decision_made
```

See [architecture.md](./architecture.md) for component diagrams and ADRs.

## API Reference

### GET /api/v1/credit-decisions/{proposal_id}

Returns the current decision for a proposal.

```json
{
  "decision_id": "uuid",
  "proposal_id": "uuid",
  "outcome": "APPROVED",
  "conditions": [
    {"type": "MAX_AMOUNT_REDUCED", "detail": "Approved for R$45,000 (requested R$50,000)"}
  ],
  "policy_evaluation_ref": "uuid",
  "decided_at": "2026-03-29T11:00:00Z",
  "overrides": []
}
```

### POST /api/v1/credit-decisions/{id}/override

Submit a manual override. Requires `X-Analyst-Id` header and appropriate role.

```json
{
  "new_outcome": "APPROVED",
  "reason_code": "STRONG_COLLATERAL",
  "justification": "Vehicle value exceeds loan by 2x, compensating for marginal DTI ratio.",
  "conditions": []
}
```

Returns `200` on success, `403` if the analyst lacks override authority, `409` if the decision has already been finalised.

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `DATABASE_URL` | JDBC PostgreSQL connection | — |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka brokers | `localhost:9092` |
| `MANUAL_REVIEW_THRESHOLD` | Policy score range routed to review | `CONDITIONAL` |
| `OVERRIDE_APPROVAL_REQUIRED_ROLE` | Role needed to override to APPROVED | `SENIOR_ANALYST` |
| `DECISION_IDEMPOTENCY_WINDOW_HOURS` | Window for dedup of duplicate events | `1` |

## Deployment

Standard EKS Helm deployment in the `credit-analysis` namespace.

```bash
helm upgrade --install credit-decision-service ./charts/credit-decision-service \
  --namespace credit-analysis \
  --values values-production.yaml
```

**Replicas:** 2-4 (HPA on CPU). **Resources:** 512Mi / 250m (request), 1Gi / 500m (limit).

The service is stateless aside from the database. Rolling deployments with zero downtime are standard.

## Monitoring

Three dashboards cover this service:

1. **Credit Decision Overview** — decision volume by outcome, latency, daily approval rate trends
2. **Manual Review Queue** — queue depth, average review time, analyst throughput
3. **Override Audit** — override volume, reason code distribution, approval chain compliance

**Critical alerts:**
- `decision_error_rate_high` — processing errors > 1% in 5 min
- `manual_review_queue_growing` — queue > 100 items
- `approval_rate_anomaly` — approval rate deviates > 15% from weekly average

## Known Issues

1. **Queue starvation during peak hours** — When loan application volume spikes (typically Monday mornings and month-end), the manual review queue can grow faster than analysts process it. Current mitigation: Slack alerting at queue depth 50. Long-term plan: auto-escalation rules that narrow the MANUAL_REVIEW band during peaks (CRED-2401).

2. **Override audit gap** — Overrides made within the first 500ms after the original decision can create a race condition where `credit_decision_made` is published before the override is persisted. This is cosmetic (the override event follows shortly), but it confuses real-time dashboards. Fix in progress (CRED-2388).

3. **No batch decision replay** — There is no mechanism to replay decisions for a set of proposals (e.g., after a policy rule correction). Replaying requires manually re-publishing `credit_policy_evaluated` events. A dedicated replay endpoint is planned.
