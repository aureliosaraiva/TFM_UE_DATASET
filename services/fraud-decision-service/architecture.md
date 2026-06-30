# Fraud Decision Service (svc-17)

**Domain:** Fraud
**Stack:** Kotlin / Spring Boot
**Database:** PostgreSQL (`fraud_decision_db`)
**Deployment:** AWS EKS

## Component Overview

Where the Fraud Scoring Service quantifies risk, the Fraud Decision Service acts on it. This service takes the fraud score and applies business rules to render a decision: approve, reject, or send to manual review. It also provides a backoffice interface for fraud analysts to override automated decisions when human judgment is warranted.

The separation between scoring and decisioning is intentional. Scores are objective model outputs; decisions incorporate business context, regulatory constraints, and operational capacity. A fraud score of 65 might mean "reject" during a fraud spike but "manual review" during normal operations, depending on the active rule configuration.

### Key Components

**Decision Rules Engine:** A table-driven rules engine that maps fraud score ranges to outcomes. Rules are organized by product type and can be adjusted by the fraud operations team through the backoffice without code changes. Example rules:
- Score 0-30: `APPROVE` (low risk)
- Score 31-65: `MANUAL_REVIEW`
- Score 66-100: `REJECT` (high risk)

Additional rules can incorporate non-score factors: "if device fingerprint shows emulator, always reject regardless of score."

**Backoffice Override API:** REST endpoints that allow authorized fraud analysts to override any automated decision. Overrides require a reason code and free-text justification. All overrides are logged immutably in a `decision_audit` table with the analyst's identity, timestamp, original decision, and override decision.

**Decision Publisher:** After a decision is rendered (whether automated or manually overridden), a `FraudDecisionRendered` event is published to SNS. The Proposal Service consumes this to advance the proposal through the fraud gate.

**Metrics Collector:** Tracks decision distribution (approve/reject/review rates), override frequency, and time-to-decision for monitoring dashboards. These metrics feed into the fraud team's operational reporting.

## Data Flow

1. `FraudScoreComputed` event arrives from the Fraud Scoring Service.
2. The service loads the applicable decision rules based on the proposal's product type.
3. Rules are evaluated against the fraud score and any supplementary signals.
4. The automated decision is persisted in `fraud_decisions` along with the rule version that produced it.
5. If the decision is `MANUAL_REVIEW`, the proposal appears in the fraud analyst backoffice queue.
6. Whether automated or after manual review, the final decision is published as `FraudDecisionRendered`.

Manual review has an SLA of 4 hours. If no analyst acts within that window, an escalation workflow triggers.

## Integration Patterns

- **Inbound (async):** SQS subscription to `FraudScoreComputed` events.
- **Inbound (sync):** Backoffice REST API for manual review actions and decision queries.
- **Outbound (async):** `FraudDecisionRendered` events to SNS, consumed by `proposal-service`.
- **PostgreSQL:** Decision records, rule configurations, and override audit trail.

## ADR-001: Separation of Scoring and Decisioning

**Context:** The initial design combined fraud scoring and decision-making in a single service. However, the fraud team needed to adjust decision thresholds frequently (sometimes daily during fraud campaigns) without risking changes to the scoring model or its infrastructure.

**Decision:** Split fraud scoring (ML model, feature engineering) and fraud decisioning (business rules, manual review) into separate services.

**Consequences:**
- The fraud operations team can adjust decision rules independently of the ML team's model deployment cycle.
- Two services to maintain instead of one, but the separation of concerns justifies the overhead.
- Clear accountability: the ML team owns the score's accuracy; the fraud operations team owns the decision thresholds' appropriateness.

## ADR-002: Immutable Decision Audit with Override Tracking

**Context:** Financial regulators require that every fraud decision be auditable, including any manual overrides. The team needed to ensure that overrides could not silently alter the audit trail.

**Decision:** The `decision_audit` table is append-only. The original automated decision is always preserved. Overrides insert a new row linked to the original, creating a chain of decisions. Neither the original nor any override can be modified or deleted. The table is backed up to S3 daily with write-once retention.

**Consequences:**
- Complete, tamper-resistant audit trail for regulatory compliance.
- The backoffice always shows both the automated decision and any overrides, providing full transparency.
- Table growth is manageable: each proposal generates 1-2 decision records at most.
