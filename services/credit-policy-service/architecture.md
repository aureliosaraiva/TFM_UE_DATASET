# Credit Policy Service (svc-21)

**Domain:** Credit
**Stack:** Kotlin / Spring Boot
**Database:** PostgreSQL (`credit_policy_db`)
**Deployment:** AWS EKS

## Component Overview

Credit policy is the set of rules that governs who gets a loan, how much, at what rate, and under what conditions. The Credit Policy Service is the system-of-record for these rules and the engine that evaluates them. It sits between the Credit Scoring Engine (which quantifies risk) and the Credit Decision Service (which renders the final verdict), translating institutional lending policy into executable logic.

What makes this service architecturally interesting is its versioning model. Lending policies change frequently -- regulatory updates, market conditions, risk appetite adjustments, new product launches -- and every change must be traceable. The service ensures that any proposal can be evaluated against the exact policy version that was active at the time, even years after the fact.

### Components

**Policy Repository:** PostgreSQL tables that define policy rules organized hierarchically:
- **Policy Set:** A named collection of rules tied to a product type (e.g., "Vehicle Secured Loan v4.2").
- **Policy Rule:** An individual condition with an operator, threshold, and action. Examples: "minimum credit score: 450," "maximum LTV: 80%," "minimum verified income: R$2,000."
- **Policy Version:** Every modification to a policy set creates a new version. Versions are immutable once published. A policy set has exactly one "active" version at any time.

**Evaluation Engine:** Given a proposal's credit score, verified income, collateral value, and other attributes, the engine loads the active policy version for the proposal's product type and evaluates each rule. The output is a structured policy result: `ELIGIBLE`, `INELIGIBLE`, or `CONDITIONAL` (eligible if certain conditions are met, e.g., "requires additional collateral").

For `ELIGIBLE` proposals, the engine also computes policy-derived parameters: maximum approved amount, applicable interest rate band, and required collateral coverage ratio.

**Policy Admin API:** REST endpoints for the credit policy team to create, modify, publish, and deactivate policy versions. Changes go through a draft/review/publish workflow. A draft can be tested against historical proposals (dry-run mode) before publication.

**Policy Event Publisher:** Publishes `PolicyEvaluationCompleted` events to SNS after each evaluation.

## Data Flow

1. `CreditScoreComputed` event triggers policy evaluation.
2. The service fetches the proposal's attributes (score, verified income, LTV, collateral type) from the event payload and, if needed, from the Proposal Service API.
3. The active policy version for the proposal's product type is loaded.
4. Rules are evaluated. Each rule produces a pass/fail result with an explanation.
5. The aggregate result, individual rule outcomes, policy version ID, and computed parameters are persisted.
6. `PolicyEvaluationCompleted` event is published to SNS.

## Integration Patterns

- **Inbound (async):** SQS subscription to `CreditScoreComputed` events.
- **Inbound (sync):** REST API for policy administration and dry-run evaluation.
- **Outbound (async):** `PolicyEvaluationCompleted` events consumed by `credit-decision-service`.
- **Outbound (sync):** Calls `proposal-service` API for supplementary proposal data.

## ADR-001: Versioned Policy Rules in Database vs. Code

**Context:** Policy rules were initially coded in Kotlin as conditional statements. Every rule change required engineering effort, code review, and deployment. The credit policy team, who are domain experts but not engineers, could not iterate independently.

**Decision:** Store policy rules as structured data in PostgreSQL. The evaluation engine interprets these rules at runtime. Complex rules that cannot be expressed as simple conditions (e.g., tiered interest rate calculations) are implemented as named "policy functions" in code, referenced by the rule record.

**Consequences:**
- The policy team can create and modify most rules through the admin API without engineering involvement.
- Policy changes take effect immediately upon version publication, no deployment required.
- The hybrid approach (data + code) means some rules still require engineering work, but these are the minority (~15% of rules).

## ADR-002: Draft/Review/Publish Workflow for Policy Changes

**Context:** An early incident occurred when a policy team member accidentally set the minimum credit score to 0 instead of 400, causing a batch of high-risk proposals to pass policy evaluation. The team needed a safety mechanism.

**Decision:** Implement a three-stage workflow: Draft (create/edit rules), Review (dry-run against the last 1,000 proposals to see impact), Publish (activate the new version). Publishing requires approval from a second policy team member (four-eyes principle).

**Consequences:**
- Impact of policy changes is visible before they go live, preventing accidents.
- The four-eyes principle adds a human gate, reducing the risk of unintended changes.
- Slight delay in policy change deployment (requires a second person to approve), but this is acceptable for a control that governs lending decisions.
