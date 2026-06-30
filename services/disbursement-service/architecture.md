# Architecture: disbursement-service

## Component Diagram

```
                     +----------------------------+
                     | registry-tracker-service   |
                     +-------------+--------------+
                                   |
                      registry_completed (Kafka)
                                   |
                                   v
+------------------------------------------------------------------------+
|                      disbursement-service                              |
|                                                                        |
|  +----------------+    +------------------------+                      |
|  | EventConsumer  |--->| DisbursementService     |                     |
|  | (Kafka)        |    +---+--------+---------+-+                      |
|  +----------------+        |        |         |                        |
|                            v        v         v                        |
|  +---------------------+ +--------+ +--------+ +-----------------+    |
|  | DisbursementController| |RuleEngine| |Approval| | EventPublisher|   |
|  | (REST API)           | |        | |Service | | (Kafka)        |    |
|  | + Backoffice APIs    | +--------+ +--------+ +-----------------+    |
|  +---------------------+                                              |
|                                                                        |
|                          +------------------+                          |
|                          | PostgreSQL       |                          |
|                          | disbursement_db  |                          |
|                          +------------------+                          |
+------------------------------------------------------------------------+
                                   |
                      disbursement_requested (Kafka)
                                   |
                                   v
                     +----------------------------+
                     | bank-transfer-integration  |
                     +----------------------------+
```

## Data Flow

1. **Registration Completed** -- The service consumes a `registry_completed` event from the `credityflow.registry` Kafka topic, indicating that the lien has been successfully registered.
2. **Order Creation** -- A `DisbursementOrder` is created with contract, borrower, and amount data extracted from the event and enriched from the contract store.
3. **Rule Evaluation** -- The `RuleEngine` evaluates all active `DisbursementRule` records against the order. Key rules include:
   - **Amount threshold:** Orders > R$500,000 require manual approval.
   - **Blackout period:** Orders created between 22:00-06:00 are queued until the next business window.
4. **Approval Workflow** -- If manual approval is required, the order status is set to `PENDING_APPROVAL`. The backoffice team reviews and approves/rejects via the REST API.
5. **Disbursement Requested** -- Once approved (or if no approval is needed), a `disbursement_requested` event is published to the `credityflow.disbursement` topic.

## Integrations

| System | Direction | Protocol | Purpose |
|---|---|---|---|
| registry-tracker-service | Inbound | Kafka event | Receives registry_completed to trigger disbursement |
| bank-transfer-integration | Outbound | Kafka event | Sends disbursement_requested for fund transfer |
| backoffice-bff | Inbound | REST API | Backoffice team manages approvals |
| PostgreSQL (disbursement_db) | Internal | JDBC | Persists orders, approvals, and rules |

## Architecture Decision Records

### ADR-035-1: Database-Driven Disbursement Rules

**Status:** Accepted
**Context:** Disbursement business rules (approval thresholds, blackout periods, etc.) change periodically as the business evolves. Hardcoding these rules requires code changes and redeployment for each adjustment.
**Decision:** Store disbursement rules in the `disbursement_rules` table with a flexible JSONB parameters column. The `RuleEngine` loads active rules at runtime and evaluates them against each disbursement order. Rules are cached in memory with a 5-minute TTL for performance.
**Consequences:** The operations team can adjust rules without engineering involvement. However, invalid rule configurations can cause unexpected behavior. A validation layer ensures rules are syntactically correct before activation. Rule changes are audit-logged.

### ADR-035-2: Manual Approval for High-Value Disbursements

**Status:** Accepted
**Context:** Regulatory and risk management requirements mandate human review for disbursements above a certain threshold. The current threshold is R$500,000, but this may change.
**Decision:** Implement a manual approval workflow integrated with the backoffice-bff. Orders exceeding the configurable threshold are set to `PENDING_APPROVAL` status. The backoffice team reviews the order details, collateral, and documentation before approving or rejecting. Approval decisions are recorded with the approver's identity and reasoning for audit purposes.
**Consequences:** The approval step introduces latency proportional to backoffice team availability. SLA monitoring is critical. The threshold is configurable via the disbursement rules mechanism (ADR-035-1), allowing adjustment without code changes.
