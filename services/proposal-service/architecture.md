# Proposal Service (svc-04)

**Domain:** Onboarding
**Stack:** Kotlin / Spring Boot
**Database:** PostgreSQL (`proposal_db`)
**Deployment:** AWS EKS

## Component Overview

At the core of CredityFlow's lending pipeline sits the Proposal Service -- the orchestrator that tracks a loan proposal from its initial creation through to final decisioning. It does not perform any business logic like fraud checks or credit scoring itself; instead, it maintains a state machine that reacts to events from across the platform and exposes the current proposal status to the frontend and backoffice applications.

The service is organized into the following components:

- **State Machine Engine** -- implements the proposal lifecycle as a finite state machine with states including `CREATED`, `DOCUMENTS_PENDING`, `DOCUMENTS_COMPLETE`, `BIOMETRICS_PENDING`, `BIOMETRICS_COMPLETE`, `FRAUD_ANALYSIS`, `CREDIT_ANALYSIS`, `APPRAISAL`, `APPROVED`, `REJECTED`, and `CANCELLED`. Transitions are triggered exclusively by domain events.
- **Event Consumer Layer** -- a set of SQS listeners subscribed to topics from Documents, Biometrics, Fraud, Credit, and Appraisal domains. Each listener maps an inbound event to a state transition command.
- **Query API** -- REST endpoints for retrieving proposal status, history, and metadata. Used heavily by the customer-facing app and the backoffice dashboard.
- **Notification Dispatcher** -- on certain state transitions (e.g., approval, rejection), the service publishes a `ProposalStatusChanged` event to SNS, which triggers downstream notification workflows (email, SMS, push).

The `proposal_db` schema centers on a `proposals` table with a `current_state` column and a `proposal_state_history` append-only audit table that records every transition with timestamp, source event, and actor.

## Data Flow

A proposal begins when the onboarding frontend calls the creation endpoint after customer registration is confirmed (via `CustomerCreated` event or direct API call). From that point forward, the proposal moves through states reactively:

1. `DocumentsValidated` event from Document Validation Service triggers transition from `DOCUMENTS_PENDING` to `DOCUMENTS_COMPLETE`.
2. `BiometricsVerified` event from Biometrics Capture Service advances the proposal past biometric verification.
3. Fraud and credit domain events arrive in sequence (or parallel, depending on configuration), each advancing the state machine.
4. The final `CreditDecisionRendered` or `AppraisalCompleted` event triggers the terminal state.

All transitions are idempotent. Receiving the same event twice does not corrupt the state machine; the duplicate is logged and discarded.

## Integration Patterns

The Proposal Service is the most heavily integrated service in the platform. It consumes events from at least 8 upstream services and produces events consumed by 4+ downstream services.

- **Inbound (async):** SQS queues subscribed to SNS topics from `customer-service`, `document-validation-service`, `biometrics-capture-service`, `fraud-decision-service`, `credit-decision-service`, and `appraisal-orchestrator-service`.
- **Outbound (async):** `ProposalStatusChanged` events to SNS, consumed by notification service, analytics, and the backoffice event stream.
- **Synchronous (REST):** Query-only. The service does not expose mutation endpoints beyond the initial proposal creation. All subsequent state changes come through events.

## ADR-001: Event-Driven State Machine over Orchestration Saga

**Context:** Two patterns were evaluated for managing the proposal lifecycle: (1) a centralized saga orchestrator that calls each service in sequence and handles compensations, and (2) a choreography-based approach where each service emits events and the proposal service passively tracks state.

**Decision:** Adopt the choreography approach with the proposal service acting as a passive state tracker rather than an active orchestrator. Each domain service is autonomous and publishes completion events; the proposal service simply listens and transitions.

**Consequences:**
- Loose coupling: adding a new step (e.g., a new fraud signal) requires only a new event subscription, not changes to orchestration logic.
- Harder to debug end-to-end flows; mitigated by the `proposal_state_history` audit trail and distributed tracing with OpenTelemetry.
- No centralized compensation logic; each domain service must handle its own rollback scenarios.

## ADR-002: Append-Only State History for Audit Compliance

**Context:** Regulatory requirements mandate a full audit trail of every decision point in the lending process. The team debated whether to rely on database triggers, application-level event sourcing, or a simpler append-only history table.

**Decision:** Use an append-only `proposal_state_history` table. Every state transition inserts a new row with the previous state, new state, triggering event ID, timestamp, and optional metadata. The `proposals` table is updated in the same transaction for fast current-state queries.

**Consequences:**
- Simple to implement and query; no event replay infrastructure needed.
- Audit-ready: regulators can inspect the full lifecycle of any proposal with a single SQL query.
- Table growth is bounded (proposals have a finite number of states), so no special archival strategy is needed for the first 2-3 years of operation.
