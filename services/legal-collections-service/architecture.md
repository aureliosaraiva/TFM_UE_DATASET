# Legal Collections Service

> svc-42 | Collections | Kotlin/Spring Boot | PostgreSQL (legal_collections_db)

## Scope

The Legal Collections Service manages the final escalation stage of the collections lifecycle. When dunning campaigns exhaust all amicable resolution paths, cases are escalated here for legal proceedings. The service tracks two primary legal instruments used in Brazilian secured lending:

1. **Protesto** (Notarial Protest) -- registering the debt with a cartorio (notary office), which impacts the debtor's credit profile and is a prerequisite for certain judicial actions.
2. **Judicial Collection** -- filing a lawsuit for debt recovery, potentially including asset seizure of the collateral.

This service does not make legal decisions autonomously. It manages the workflow, tracks deadlines, and integrates with external legal service providers.

## Components

```
+----------------------------------------------+
|          Legal Collections Service            |
|                                               |
|  +-----------------+   +------------------+   |
|  | CaseIntake      |   | ProtestoManager  |   |
|  | Controller/     |   | (cartorio        |   |
|  | EventListener   |   |  integration)    |   |
|  +-----------------+   +------------------+   |
|                                               |
|  +-----------------+   +------------------+   |
|  | JudicialCase    |   | DeadlineTracker  |   |
|  | Manager         |   | (cron-based)     |   |
|  +-----------------+   +------------------+   |
|                                               |
|  +-----------------+   +------------------+   |
|  | LegalProvider   |   | CaseRepository   |   |
|  | Adapter         |   | (PostgreSQL)     |   |
|  +-----------------+   +------------------+   |
+----------------------------------------------+
```

**CaseIntake:** Receives `EscalateToLegal` commands from the dunning service. Creates a legal case record and determines the initial legal action based on contract type and jurisdiction.

**ProtestoManager:** Manages the protesto lifecycle: submission to cartorio (via legal provider API), tracking registration confirmation, handling cancellation when payment is received. Brazilian law requires protesto cancellation within specific timeframes after payment.

**JudicialCaseManager:** Tracks judicial proceedings: petition filing, citation, hearings, judgment. Integrates with external law firm case management systems via REST APIs.

**DeadlineTracker:** A cron job running every 4 hours that checks for approaching legal deadlines (statute of limitations, response periods, cancellation obligations) and creates alerts in the case management service.

**LegalProviderAdapter:** Abstraction layer over external legal service providers. Currently integrates with one provider; designed for multi-provider support as CredityFlow scales to new jurisdictions.

## Data Flow

```
EscalateToLegal (SQS, from dunning-service)
        |
        v
  CaseIntake --> legal_collections_db (case created, status: INTAKE)
        |
        +---> ProtestoManager ---> Legal Provider API (submit protesto)
        |                                |
        |                          (async callback)
        |                                |
        |                                v
        |                    legal_collections_db (status: PROTESTED)
        |
        +---> JudicialCaseManager ---> Legal Provider API (file petition)
                                             |
                                       (status updates via polling)
                                             |
                                             v
                                   legal_collections_db (status: IN_LITIGATION)

PaymentConfirmed (SQS, from billing-service)
        |
        v
  ProtestoManager --> Legal Provider API (cancel protesto)
        |
        v
  legal_collections_db (status: RESOLVED)
```

## Integration Patterns

- **Legal Provider API:** Synchronous REST calls for submissions, asynchronous callbacks and polling for status updates. Circuit breaker with 60-second half-open window due to the provider's variable response times.
- **Billing Service:** Subscribes to `PaymentConfirmed` events to trigger protesto cancellation and case resolution. This is critical -- failure to cancel protesto after payment is a regulatory violation.
- **Case Management Service:** Publishes `LegalCaseCreated` and `LegalCaseUpdated` events so the backoffice has visibility into legal proceedings alongside other case types.
- **Audit trail:** Every state transition in a legal case is published to the audit service. Legal cases have strict auditability requirements.

## ADR

### ADR-042-001: Separate database from other collections services

**Context:** Other collections services share `collections_db`. Legal data has distinct retention requirements (10+ years for judicial records) and access control needs (restricted to legal team).

**Decision:** Use a dedicated `legal_collections_db` with separate access credentials and a longer retention policy. Row-level security restricts access to the legal operations team.

**Consequences:** Isolation of sensitive legal data. Independent backup and retention policies. Cross-domain queries (e.g., "show me all collections activity for contract X") require joining data from multiple services via the case management service, not direct DB queries.

### ADR-042-002: Protesto cancellation as a critical path with alerting

**Context:** Brazilian law mandates timely cancellation of protesto after debt payment. Failure to cancel exposes CredityFlow to legal liability and damages to the debtor.

**Decision:** Treat protesto cancellation as a critical workflow with dedicated monitoring. If a `PaymentConfirmed` event is received for a protested contract and the cancellation is not confirmed within 24 hours, an alert is escalated to the legal team and the on-call engineer.

**Consequences:** Reduces regulatory risk. Adds operational overhead for monitoring. The 24-hour alert window accounts for cartorio processing times while still being aggressive enough to catch failures.
