# Architecture: registry-orchestrator-service

## Component Diagram

```
                          +--------------------------+
                          |  contract-generation-svc |
                          +------------+-------------+
                                       |
                            contract_signed (Kafka)
                                       |
                                       v
+--------------------------------------------------------------------------+
|                   registry-orchestrator-service                          |
|                                                                          |
|  +----------------+    +------------------------+    +-----------------+ |
|  | EventConsumer  |--->| RegistrationOrchestrator|--->| EventPublisher | |
|  | (Kafka)        |    |   + SagaManager         |    | (Kafka)        | |
|  +----------------+    +----------+-------------+    +-----------------+ |
|                                   |                                      |
|  +--------------------+    +------v-------+                              |
|  | RegistrationController  | PostgreSQL   |                              |
|  | (REST API)         |    | registry_db  |                              |
|  +--------------------+    +--------------+                              |
+--------------------------------------------------------------------------+
                                       |
                          registry_requested (Kafka)
                                       |
                       +---------------+---------------+
                       v                               v
          +---------------------+         +------------------------+
          | detran-integration  |         | cartorio-integration   |
          +---------------------+         +------------------------+
```

## Data Flow

1. **Contract Signed** -- The service consumes a `contract_signed` event from the `credityflow.contracts` Kafka topic.
2. **Registration Created** -- A `RegistrationRequest` record is persisted in PostgreSQL, and a `SagaState` is initialized with status `STARTED`.
3. **Routing Decision** -- The orchestrator inspects the collateral type: `VEHICLE` routes to DETRAN, `PROPERTY` routes to cartorio.
4. **Event Published** -- A `registry_requested` event is published to the `credityflow.registry` topic with the routing channel encoded in the event header.
5. **Saga Tracking** -- The SagaState is updated to `IN_PROGRESS`. Downstream completion or failure events will transition it to `COMPLETED` or trigger compensation.

## Integrations

| System | Direction | Protocol | Purpose |
|---|---|---|---|
| contract-generation-service | Inbound | Kafka event | Receives contract_signed to trigger registration |
| detran-integration-service | Outbound | Kafka event | Sends registry_requested for vehicle liens |
| cartorio-integration-service | Outbound | Kafka event | Sends registry_requested for property liens |
| PostgreSQL (registry_db) | Internal | JDBC | Persists registration requests and Saga state |

## Architecture Decision Records

### ADR-031-1: Saga Pattern for Registration Orchestration

**Status:** Accepted
**Context:** Lien registration involves multiple external systems (DETRAN, cartorio) with different latencies and failure modes. A simple request-response model would not provide the reliability or traceability required.
**Decision:** Implement the Saga pattern with choreography-based coordination via Kafka events. The orchestrator manages Saga state in PostgreSQL and publishes routing events. Downstream services publish completion or failure events that the orchestrator consumes to advance or compensate the Saga.
**Consequences:** Increased complexity in state management, but provides clear traceability, automatic retry capability, and a foundation for compensation logic. Each step is independently recoverable.

### ADR-031-2: Routing by Collateral Type

**Status:** Accepted
**Context:** Different collateral types require registration with different government bodies. Vehicles require DETRAN registration; properties require cartorio registration.
**Decision:** Encode the routing decision in the orchestrator based on collateral type from the contract data. The routing channel is included as a header in the `registry_requested` event, and downstream consumers filter by channel.
**Consequences:** Adding new collateral types requires updating the routing logic in this service. The approach keeps downstream services simple and focused on a single integration each.
