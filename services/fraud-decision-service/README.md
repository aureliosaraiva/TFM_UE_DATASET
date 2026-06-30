# Fraud Decision Service

> **Service name:** `fraud-decision-service` — last stop in the fraud pipeline before credit assessment.

## Overview

Where the fraud-scoring-service answers "how risky is this proposal?", the `fraud-decision-service` answers "what should we do about it?". It takes the fraud risk score, applies configurable business rules, and renders a final decision: **APPROVED** (proceed with the loan), **DENIED** (block it), or **MANUAL_REVIEW** (a human needs to look at this).

The service also provides a backoffice override mechanism. When a proposal lands in MANUAL_REVIEW, a fraud analyst can examine the details and override the automated decision. Every override is recorded with the analyst's identity and reasoning for compliance and audit purposes.

This is a Kotlin / Spring Boot service backed by PostgreSQL (`fraud_decisions_db`). It is the last stop in the fraud pipeline before the credit assessment phase begins.

## Getting Started

### Prerequisites

- JDK 17+
- Docker and Docker Compose (PostgreSQL, Kafka)

### Running Locally

```bash
docker-compose up -d postgres kafka
./gradlew bootRun --args='--spring.profiles.active=local'
```

The local profile seeds the database with default decision rules on first startup:

| Rule | Condition | Decision |
|---|---|---|
| Auto-approve | Score <= 250 | APPROVED |
| Low-medium review | Score 251-500 | APPROVED (with flag) |
| High-risk review | Score 501-750 | MANUAL_REVIEW |
| Auto-deny | Score > 750 | DENIED |
| Watchlist mandatory review | Any watchlist match | MANUAL_REVIEW |

These rules can be modified at runtime via the database. No deploy required.

### Tests

```bash
./gradlew test
./gradlew integrationTest
```

The integration tests cover the full decision flow: publishing a `fraud_scored` event and asserting the correct decision and downstream events are produced. Override flows are also covered.

## Architecture

Standard hexagonal layout:

- **Inbound adapters** -- Kafka consumer for `fraud_scored` events, REST controller for queries and overrides
- **Domain layer** -- `FraudDecision`, `FraudOverride`, `DecisionRule` entities and the `DecisionEngine`
- **Outbound adapters** -- JPA repositories, Kafka producer

The `DecisionEngine` is the core component. It loads active rules, sorts them by priority, and evaluates the fraud score against each rule. The first matching rule determines the decision. This means rule priority matters -- higher-priority rules are checked first.

There is a deliberate separation between the automated decision path (event-driven) and the override path (REST API). Both paths converge on the same `FraudDecision` entity, but the override path requires analyst credentials and writes an additional `FraudOverride` audit record.

### Decision Events

The service publishes three types of events depending on the outcome:

- `fraud_decision_made` -- Always published, regardless of outcome. Contains the full decision record.
- `fraud_cleared` -- Published only when the decision is APPROVED (automated or via override). This is what the credit pipeline listens for.
- `fraud_flagged` -- Published when the decision is DENIED or MANUAL_REVIEW. May trigger notifications or case creation.

## API Reference

### GET /api/v1/fraud-decisions/{proposal_id}

Retrieve the fraud decision for a proposal.

**Response (200):**
```json
{
  "proposalId": "proposal-uuid",
  "decision": "MANUAL_REVIEW",
  "fraudScore": 620,
  "riskLevel": "HIGH",
  "triggeredRule": "high-risk-review",
  "decidedAt": "2025-09-15T11:21:00Z",
  "override": null
}
```

If the decision has been overridden:
```json
{
  "proposalId": "proposal-uuid",
  "decision": "APPROVED",
  "fraudScore": 620,
  "riskLevel": "HIGH",
  "triggeredRule": "high-risk-review",
  "decidedAt": "2025-09-15T11:21:00Z",
  "override": {
    "previousDecision": "MANUAL_REVIEW",
    "newDecision": "APPROVED",
    "analystId": "analyst-042",
    "reason": "Customer is a returning borrower with clean history. Device reuse flag was due to shared household device.",
    "overriddenAt": "2025-09-15T14:05:00Z"
  }
}
```

### POST /api/v1/fraud-decisions/{id}/override

Override a fraud decision. Restricted to authenticated backoffice analysts.

**Request:**
```json
{
  "newDecision": "APPROVED",
  "reason": "Verified with customer via phone. False positive on device reuse."
}
```

**Response (200):** Returns the updated decision object.

The `reason` field is mandatory and must be at least 20 characters. This is enforced to ensure meaningful audit trails.

## Configuration

| Property | Description | Default |
|---|---|---|
| `spring.datasource.url` | PostgreSQL connection | `jdbc:postgresql://localhost:5432/fraud_decisions_db` |
| `kafka.bootstrap-servers` | Kafka brokers | `localhost:9092` |
| `kafka.topic.fraud-scored` | Input topic | `fraud.scored` |
| `kafka.topic.fraud-decision` | Output topic | `fraud.decision.made` |
| `kafka.topic.fraud-cleared` | Cleared topic | `fraud.cleared` |
| `kafka.topic.fraud-flagged` | Flagged topic | `fraud.flagged` |
| `decision.rules.cache-ttl-seconds` | Rule cache TTL | `30` |

## Deployment

Deployed on AWS EKS.

- **Replicas:** 2 in staging, 3 in production
- **Resources:** 512Mi memory, 250m CPU
- **Health checks:** `/actuator/health`
- **Secrets:** Database credentials and backoffice auth tokens via Kubernetes secrets

The service is stateless and scales horizontally. Kafka partition assignment handles consumer distribution across replicas.

## Monitoring

### Key Metrics

- **`fraud_decisions_total` by outcome** -- The approve/deny/review split. Typical distribution: ~75% APPROVED, ~15% MANUAL_REVIEW, ~10% DENIED. Significant deviations warrant investigation.
- **`fraud_overrides_total` by direction** -- How often analysts override decisions. If the override rate is high (>50% of manual reviews get overridden), the decision rules probably need recalibration.
- **`manual_review_backlog_size`** -- Number of decisions in MANUAL_REVIEW that have not been overridden. If this grows faster than analysts can process, it creates a bottleneck in loan processing.

### Alerts

- Denial rate > 30% for 1 hour -- Rule misconfiguration or fraud wave.
- Manual review backlog > 100 -- Analysts are overwhelmed or offline.
- Override rate > 50% -- Rules are too aggressive.
- Decision latency p99 > 2 seconds.

### Dashboard

**Fraud Decision Overview** -- Decision outcome breakdown (pie), override rates (line), manual review queue depth (gauge), daily decision volume (bar).

## Known Issues

1. **No four-eyes principle on overrides.** A single analyst can override any decision. The compliance team has requested a dual-approval workflow for DENIED-to-APPROVED overrides, but it has not been implemented yet.

2. **Rule changes take effect immediately.** There is no staging or preview for rule changes. If someone misconfigures a rule (e.g., sets the auto-deny threshold to 100 instead of 1000), it affects all subsequent decisions instantly. Adding a "draft" rule state with a preview against recent proposals is planned.

3. **Manual review queue has no SLA enforcement.** Proposals in MANUAL_REVIEW just sit there until an analyst acts. There is no escalation or auto-decision after a timeout. This can delay loan disbursement for customers caught in the queue over weekends.

4. **Event publishing is not transactional.** The decision is persisted and then events are published. If Kafka is down, the decision exists in the database but downstream services are not notified. A transactional outbox pattern (like lead-intake-service uses) would solve this but has not been adopted here yet.

5. **Decision rules are global.** The same rules apply to all loan products and customer segments. The team has discussed adding product-specific rule sets, but the current volume does not justify the complexity.
