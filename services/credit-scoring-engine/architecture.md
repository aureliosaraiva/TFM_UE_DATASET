# Credit Scoring Engine (svc-20)

**Domain:** Credit
**Stack:** Python / FastAPI
**Database:** PostgreSQL (`credit_scoring_db`)
**Deployment:** AWS EKS (compute-optimized node group)

## Component Overview

The Credit Scoring Engine is the quantitative core of CredityFlow's credit analysis pipeline. It ingests structured data about a loan applicant -- bureau report, verified income, proposal details, and historical performance data -- and produces a credit risk score using an XGBoost gradient boosting model. This score directly influences the lending decision: interest rate, approved amount, and whether the proposal proceeds at all.

The service implements a CQRS (Command Query Responsibility Segregation) pattern. Write operations (score computation) are triggered exclusively through events, never through REST calls. Read operations (score retrieval, model explainability queries) are served via REST. This separation ensures that scoring throughput is not affected by query load, and vice versa.

### Architectural Components

**Event Intake (Command Side):** SQS consumers listen for three prerequisite events: `CreditBureauReportReady`, `IncomeVerificationCompleted`, and `ProposalEnriched` (from proposal-service, containing loan amount, term, collateral type). A correlation engine groups these events by proposal ID and triggers scoring when all prerequisites are met.

**Feature Pipeline:** Transforms raw event data into model features. This is a non-trivial component -- the XGBoost model uses 87 features, including:
- Bureau features: credit score, number of active debts, oldest account age, utilization ratio.
- Income features: verified income, income stability indicator, debt-to-income ratio.
- Proposal features: loan-to-value ratio, term in months, collateral type.
- Derived features: interaction terms, historical default rates for similar profiles.

The feature pipeline is versioned alongside the model to ensure consistency.

**XGBoost Inference:** The trained model is loaded from S3 at startup. Inference produces a probability of default (PD) in the range [0.0, 1.0], which is mapped to a credit score on a 300-900 scale (following Brazilian market convention). Model inference takes 5-15ms per request on CPU.

**Explainability Module:** Uses SHAP (SHapley Additive exPlanations) values to decompose each score into feature-level contributions. This is stored alongside the score and is accessible via the REST API. Regulators and backoffice analysts can see exactly why a particular score was assigned.

**Score Store (Query Side):** PostgreSQL stores the final score, PD, feature vector, SHAP values, model version, and timestamp. The REST API serves queries against this store without touching the scoring pipeline.

## Data Flow

**Write path (events):**
1. Prerequisite events arrive and are correlated by proposal ID.
2. Feature pipeline constructs the 87-feature vector.
3. XGBoost model computes PD and credit score.
4. SHAP values are calculated for explainability.
5. All outputs are persisted to PostgreSQL.
6. `CreditScoreComputed` event is published to SNS.

**Read path (REST):**
- `GET /scores/{proposal_id}` -- returns the latest score, PD, and SHAP summary.
- `GET /scores/{proposal_id}/explain` -- returns detailed SHAP values and feature contributions.
- `GET /models/current` -- returns metadata about the active model version.

The two paths share the PostgreSQL database but operate independently. The write path can be scaled by adding SQS consumer replicas; the read path by adding API replicas behind the load balancer.

## Integration Patterns

- **Inbound (async, command):** SQS queues for `CreditBureauReportReady`, `IncomeVerificationCompleted`, `ProposalEnriched`.
- **Inbound (sync, query):** REST API for score retrieval and explainability.
- **Outbound (async):** `CreditScoreComputed` event to SNS, consumed by `credit-policy-service` and `credit-decision-service`.
- **S3:** Model artifact storage, feature pipeline configuration.

## ADR-001: CQRS Pattern for Scoring Workload

**Context:** Early versions of the service exposed a synchronous `/score` POST endpoint. Under load, backoffice users querying scores competed with incoming scoring requests for database connections and CPU. Response times became unpredictable.

**Decision:** Adopt CQRS. All score computation is triggered by events (command side). All score queries go through REST (query side). The two sides share the database but have separate connection pools and can be scaled independently.

**Consequences:**
- Predictable latency on both sides: scoring throughput is governed by SQS consumer concurrency; query latency is governed by API replicas and database read performance.
- Slightly more complex deployment (two distinct scaling dimensions), but Kubernetes HPA handles this transparently.
- No eventual consistency issues because scores are immutable once computed.

## ADR-002: XGBoost over Linear Models and Deep Learning

**Context:** The ML team evaluated logistic regression (simple, interpretable), XGBoost (powerful, reasonably interpretable), and a neural network (highest potential accuracy). The evaluation used 500K historical proposals with known outcomes.

**Decision:** XGBoost. It achieved a Gini coefficient of 0.68 (vs. 0.52 for logistic regression and 0.71 for the neural network) while maintaining interpretability through SHAP values and running on CPU without GPU infrastructure.

**Consequences:**
- Strong predictive performance without GPU costs.
- SHAP-based explainability satisfies regulatory requirements for scoring transparency.
- The 3-point Gini gap vs. the neural network translates to approximately R$200K/year in additional expected losses at current portfolio volume. This gap will be reassessed annually as the portfolio grows.
