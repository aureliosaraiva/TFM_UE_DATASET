# svc-39: Dunning Service

## Domain Context

Part of the **Collections** domain. The Dunning Service manages multi-step dunning campaigns -- automated sequences of escalating actions taken when a borrower misses a payment. It sits between the billing service (which detects overdue installments) and the operational services that execute contact attempts, negotiation offers, and legal proceedings.

**Stack:** Kotlin / Spring Boot 3.x / PostgreSQL (`dunning_db`)

---

## Component Overview

The service is built around the concept of a **Dunning Campaign**, a state machine that progresses through configured steps based on time delays and borrower responses.

### Campaign State Machine

```
INITIATED --> REMINDER_SENT --> CONTACT_SCHEDULED --> NEGOTIATION_OFFERED
                                                            |
                                              +-------------+-------------+
                                              v                           v
                                      NEGOTIATION_ACCEPTED         ESCALATED_TO_LEGAL
                                              |                           |
                                              v                           v
                                          RESOLVED                   LEGAL_PENDING
```

### Internal Components

- **CampaignOrchestrator**: Listens for `InstallmentOverdue` events. Creates a new dunning campaign for each overdue contract (or appends to an existing active campaign). Determines the initial step based on days-past-due and customer risk tier.
- **StepScheduler**: A time-based scheduler that advances campaigns through their configured steps. Uses a database-backed job queue (not Quartz -- simple polling with advisory locks for leader election).
- **ActionDispatcher**: When a step is reached, dispatches commands to the appropriate service: sends `SendReminder` to notification-service, `ScheduleContact` to collections-contact-service, or `GenerateOffer` to negotiation-engine-service.
- **ResponseHandler**: Listens for outcome events (`ContactCompleted`, `NegotiationAccepted`, `PaymentConfirmed`) and transitions the campaign accordingly.
- **CampaignConfigRepository**: Stores campaign templates (step sequences, delays, escalation rules) per product type. Configurable via backoffice without code changes.

### Database Model

The `dunning_db` schema centers on three tables:
- `campaigns` -- one per overdue contract, tracks current step and status.
- `campaign_steps` -- log of each step executed, with timestamps and outcomes.
- `campaign_configs` -- templates defining step sequences per product/risk tier.

## Data Flow

Inbound:
- `InstallmentOverdue` (from billing-service via SQS) triggers campaign creation.
- `PaymentConfirmed` (from billing-service) resolves active campaigns.
- `ContactCompleted`, `NegotiationAccepted/Rejected` (from respective services) advance campaign state.

Outbound:
- `SendReminder` command to notification-service (via SQS).
- `ScheduleContact` command to collections-contact-service (via SQS).
- `GenerateOffer` command to negotiation-engine-service (via SQS).
- `EscalateToLegal` command to legal-collections-service (via SQS).

## Integration Patterns

The dunning service is a **choreography coordinator** -- it does not call other services synchronously. All interactions are asynchronous command/event messages through SNS/SQS. This makes the service resilient to downstream outages; if the notification service is down, the command sits in the queue until it recovers.

Campaign configurations are loaded at startup and cached. Changes via the backoffice API trigger a cache invalidation event.

## ADR

### ADR-039-001: Database-backed job scheduler over distributed scheduling (SQS delay / Step Functions)

**Context:** Campaign steps need delayed execution (e.g., "send reminder 3 days after overdue"). Options included SQS delay queues (max 15 min), AWS Step Functions (cost at scale), or a database-backed scheduler.

**Decision:** Implement a simple database-backed scheduler that polls every 5 minutes for steps whose `scheduled_at <= now()`. Uses PostgreSQL advisory locks for leader election across replicas.

**Consequences:** No external dependency for scheduling. Maximum 5-minute jitter on step execution, which is acceptable for dunning timelines measured in days. Simpler operational model -- all state visible in the database. Tradeoff: polling creates constant (small) database load.

### ADR-039-002: One campaign per contract, not per installment

**Context:** A borrower might have multiple overdue installments on the same contract. Creating separate campaigns per installment would lead to the borrower receiving duplicate communications.

**Decision:** Create one campaign per contract. When additional installments become overdue on the same contract, they are attached to the existing active campaign, which may escalate its current step.

**Consequences:** Prevents communication fatigue. Simplifies the borrower's experience -- one conversation thread, one negotiation. Adds complexity to the campaign logic (must handle "installment added to active campaign" transitions).
