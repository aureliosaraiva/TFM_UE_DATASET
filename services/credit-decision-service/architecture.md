# Credit Decision Service (svc-22)

**Domain:** Credit
**Stack:** Kotlin / Spring Boot
**Database:** PostgreSQL (`credit_decision_db`)
**Deployment:** AWS EKS

## Component Overview

The Credit Decision Service is the terminal node in CredityFlow's credit analysis pipeline. After the credit score has been computed, income verified, and policy rules evaluated, this service synthesizes everything into a final credit decision. It is, in essence, the service that says "yes," "no," or "let a human decide."

The service occupies a position of significant responsibility. Its output directly determines whether a customer receives a loan, at what amount, and at what interest rate. For this reason, it is designed with extensive auditability, a robust manual override capability, and a clear separation between automated and human-assisted decisions.

### Components

**Decision Aggregator:** Collects the prerequisite inputs for a credit decision:
- Fraud decision (from svc-17): must be `APPROVED` for credit analysis to proceed.
- Credit score and PD (from svc-20).
- Policy evaluation result (from svc-21): eligibility status and computed parameters.
- Income verification result (from svc-19): verified income and verification confidence.

The aggregator holds state in PostgreSQL, tracking which inputs have been received for each proposal. A decision is rendered once all four inputs are present.

**Automated Decision Logic:** For straightforward cases (clear policy pass + low fraud risk + verified income), the decision is rendered automatically:
- Policy `ELIGIBLE` + fraud `APPROVED` = credit `APPROVED` with parameters from policy evaluation.
- Policy `INELIGIBLE` = credit `REJECTED` with specific rejection reasons.
- Policy `CONDITIONAL` or income `DISCREPANCY_DETECTED` = routed to `MANUAL_REVIEW`.

**Manual Review Interface:** A backoffice REST API that presents pending reviews to credit analysts. Analysts see the full decision context: score, SHAP explanation, policy evaluation details, income verification breakdown, and fraud signals. They can approve (optionally adjusting amount or rate within policy bounds), reject, or request additional documentation.

**Decision Record:** Every decision -- automated or manual -- is persisted as an immutable record with all inputs, the decision logic path taken, and the outcome. For manual decisions, the analyst ID, timestamp, and justification are included.

**Event Publisher:** Publishes `CreditDecisionRendered` to SNS. This is the event that the Proposal Service awaits to advance the proposal to the next stage (or terminate it).

## Data Flow

1. Four prerequisite events arrive asynchronously and are correlated by proposal ID in the aggregator.
2. Once all inputs are present, the automated decision logic evaluates the case.
3. For auto-decisions, the result is immediately persisted and published.
4. For manual review cases, the proposal enters the review queue. The decision is persisted and published only after an analyst acts.
5. `CreditDecisionRendered` event is consumed by the Proposal Service and, for approved proposals, triggers the appraisal flow.

## Integration Patterns

- **Inbound (async):** SQS queues for `FraudDecisionRendered`, `CreditScoreComputed`, `PolicyEvaluationCompleted`, and `IncomeVerificationCompleted`.
- **Inbound (sync):** Backoffice REST API for manual review and decision queries.
- **Outbound (async):** `CreditDecisionRendered` event to SNS.
- **Outbound (sync):** Calls `credit-scoring-engine` REST API to fetch SHAP explanations for the backoffice review screen.

## ADR-001: Final Decision Service vs. Distributed Decision Making

**Context:** The team considered letting each upstream service (fraud, policy, income) render its own partial decision, with the Proposal Service aggregating them into a final outcome. This would eliminate the need for a dedicated decision service but distribute decision logic across multiple services.

**Decision:** Centralize the final credit decision in a single service. All inputs are aggregated here, and the decision logic lives in one place.

**Consequences:**
- Single, auditable point where the lending decision is made. Regulators can inspect one service to understand how decisions are rendered.
- The decision service becomes a bottleneck if any upstream input is delayed. Mitigated by the timeout mechanism (if an input does not arrive within 10 minutes, the proposal is routed to manual review with available data).
- Easier to implement cross-cutting decision logic (e.g., "if fraud score is borderline AND income has a discrepancy, always reject").

## ADR-002: Manual Override with Bounded Authority

**Context:** Credit analysts need the ability to override automated decisions, but unrestricted overrides create risk. The team needed to define guardrails.

**Decision:** Overrides are bounded by policy parameters. An analyst can approve a proposal that was auto-routed to review, but the approved amount cannot exceed the policy-computed maximum. An analyst can adjust the interest rate, but only within the rate band defined by the active policy version. To approve beyond policy bounds, a senior credit officer must co-approve.

**Consequences:**
- Analysts have meaningful authority while staying within institutional risk limits.
- The two-tier approval structure (analyst within bounds, senior officer beyond bounds) balances flexibility with control.
- All overrides, including the authority level used, are part of the immutable decision record.
