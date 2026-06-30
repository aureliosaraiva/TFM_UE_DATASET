# Architecture — compliance-check-service

## Component Diagram

```
                         ┌──────────────────────────┐
  contract_generated     │  compliance-check-service │
  ─────────────────────► │                           │
  (Kafka)                │  ┌─────────────────────┐  │
                         │  │   Kafka Consumer     │  │     ┌────────────┐
                         │  │   Adapter            │──┼────►│ PostgreSQL │
                         │  └────────┬────────────┘  │     │ contract_db│
                         │           │                │     └────────────┘
                         │           ▼                │
                         │  ┌─────────────────────┐  │
                         │  │   Compliance Rule    │  │
  REST Clients           │  │   Engine (Core)      │  │
  ──────────────────►    │  └────────┬────────────┘  │
  POST /compliance/check │           │                │
  GET  /compliance/rules │           ▼                │
                         │  ┌─────────────────────┐  │
                         │  │   Event Publisher    │  │
                         │  │   Adapter            │  │
                         │  └────────┬────────────┘  │
                         └───────────┼────────────────┘
                                     │
                                     ▼
                              compliance_checked
                              (Kafka)
```

## Data Flow

1. **Ingress — Event-driven path.** The Kafka consumer adapter picks up `contract_generated` messages from the `contract.events` topic. Each message contains the full contract payload including product type, interest rates, fees, and customer details needed for validation.

2. **Ingress — Synchronous path.** Upstream services or backoffice tools can also invoke `POST /api/v1/compliance/check` directly for ad-hoc validation outside the normal pipeline flow.

3. **Processing.** The Compliance Rule Engine loads applicable rules from an in-memory cache (backed by PostgreSQL). Rules are matched by product type and effective date. Each rule is evaluated sequentially, collecting violations. The engine produces a `ComplianceResult` with a verdict of PASS (no violations) or FAIL (one or more violations).

4. **Persistence.** Every `ComplianceResult` is written to PostgreSQL for audit trail purposes. The `compliance_results` table stores contract ID, verdict, violation details as JSONB, and a timestamp. This data must be retained for 5 years per regulatory requirements.

5. **Egress.** After persistence, the service publishes a `compliance_checked` event to Kafka. Downstream, `digital-signature-service` consumes this event. Only contracts with a PASS verdict proceed to signature; FAIL results trigger a notification to the origination team.

## Integrations

| System | Direction | Protocol | Notes |
|---|---|---|---|
| contract-generation-service | Inbound | Kafka | Produces `contract_generated` events consumed by this service |
| digital-signature-service | Outbound | Kafka | Consumes `compliance_checked` events published by this service |
| PostgreSQL (contract_db) | Internal | JDBC | Stores rules, requirements, and audit records |

There are no external third-party integrations. All regulatory rules are maintained internally.

## ADR-001: Synchronous Rule Evaluation Over Async Pipeline

**Status:** Accepted

**Context:** We considered two designs for the rule engine: (a) evaluate all rules synchronously in a single request/event cycle, or (b) fan out rule evaluations as separate async tasks and aggregate later.

**Decision:** Synchronous evaluation within a single thread per contract. The rule set is small enough (currently ~120 rules) that full evaluation completes in under 200ms. The synchronous approach keeps the codebase simple, avoids partial-result complexity, and produces a single atomic verdict per contract.

**Consequences:** If the rule set grows significantly (>1,000 rules) or individual rules require external calls, this decision should be revisited. For now, the latency budget is well within the 5-second SLA.

## ADR-002: Rule Cache with TTL Instead of Event-Driven Invalidation

**Status:** Accepted

**Context:** Compliance rules change infrequently (a few times per quarter). We needed a strategy to avoid hitting the database on every check while ensuring rule updates propagate in a reasonable timeframe.

**Decision:** Use a Caffeine in-memory cache with a 5-minute TTL. On cache miss, rules are loaded from PostgreSQL. No event-driven invalidation mechanism is implemented.

**Consequences:** After a rule deployment, there is a window of up to 5 minutes where pods may evaluate contracts against stale rules. This is acceptable because rule deployments are planned events coordinated with a brief pause in contract processing. If real-time rule propagation becomes necessary, we can introduce a Kafka-based cache invalidation topic.
