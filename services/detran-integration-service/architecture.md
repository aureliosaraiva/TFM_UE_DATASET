# Architecture: detran-integration-service

## Component Diagram

```
                     +----------------------------+
                     | registry-orchestrator-svc  |
                     +-------------+--------------+
                                   |
                      registry_requested (Kafka)
                        [channel=DETRAN]
                                   |
                                   v
+------------------------------------------------------------------------+
|                    detran-integration-service                          |
|                                                                        |
|  +----------------+    +---------------------------+                   |
|  | EventConsumer  |--->| DetranRegistrationService  |                  |
|  | (Kafka)        |    +--+---------------------+--+                   |
|  +----------------+       |                     |                      |
|                           v                     v                      |
|  +--------------------+  +-----------------------+  +----------------+ |
|  | DetranController   |  | StateAdapterFactory   |  | EventPublisher | |
|  | (REST API)         |  |  +-- SP Adapter       |  | (Kafka)        | |
|  +--------------------+  |  +-- RJ Adapter       |  +----------------+ |
|                          |  +-- MG Adapter       |                     |
|                          |  +-- ...              |                     |
|                          +----------+------------+                     |
|                                     |                                  |
|                          +----------v-----------+                      |
|                          | DetranApiClient      |                      |
|                          | (mTLS HTTP client)   |                      |
|                          +----------+-----------+                      |
|                                     |              +----------------+  |
|                                     |              | PostgreSQL     |  |
|                                     |              | registry_db    |  |
|                                     |              +----------------+  |
+------------------------------------------------------------------------+
                                      |
                                      v (mTLS)
                          +-----------------------+
                          |    DETRAN APIs         |
                          | (per-state endpoints)  |
                          +-----------------------+

                      detran_lien_registered (Kafka)
                                      |
                                      v
                     +----------------------------+
                     | registry-tracker-service   |
                     +----------------------------+
```

## Data Flow

1. **Event Received** -- The service consumes `registry_requested` events from Kafka, filtered by `channel=DETRAN` header.
2. **State Resolution** -- The `StateAdapterFactory` resolves the appropriate DETRAN adapter based on the vehicle's `state_code`.
3. **API Submission** -- The adapter formats the request according to the target state's API specification and submits via `DetranApiClient` using mTLS.
4. **Status Tracking** -- The service persists the registration in PostgreSQL and polls or receives a callback from DETRAN for the final status.
5. **Completion Event** -- Once DETRAN confirms the lien, a `detran_lien_registered` event is published to Kafka.

## Integrations

| System | Direction | Protocol | Purpose |
|---|---|---|---|
| registry-orchestrator-service | Inbound | Kafka event | Receives registry_requested for vehicle liens |
| DETRAN APIs (per state) | Outbound | REST + mTLS | Submits and queries vehicle lien registrations |
| registry-tracker-service | Outbound | Kafka event | Publishes detran_lien_registered for tracking |
| PostgreSQL (registry_db) | Internal | JDBC | Persists registration data and vehicle info |

## Architecture Decision Records

### ADR-032-1: Strategy Pattern for State-Specific DETRAN Adapters

**Status:** Accepted
**Context:** Each Brazilian state operates its own DETRAN API with different endpoints, request/response formats, authentication methods, and error handling. A monolithic integration would become unmaintainable as more states are added.
**Decision:** Use the Strategy pattern with a `DetranRegistrationPort` interface and state-specific adapter implementations. A `StateAdapterFactory` resolves the correct adapter at runtime based on the vehicle's state code. Each adapter encapsulates the full API contract for its target state.
**Consequences:** Adding a new state requires implementing a new adapter and registering it in the factory. The core domain logic remains unchanged. Testing is simplified as each adapter can be tested independently with state-specific mocks.

### ADR-032-2: mTLS for DETRAN API Communication

**Status:** Accepted
**Context:** DETRAN APIs require mutual TLS (mTLS) authentication with certificates issued per financial institution. Some states also require an additional API key.
**Decision:** Configure a per-state HTTP client with mTLS using a shared keystore. Certificate management is handled outside the application via Kubernetes secrets. The `DetranApiClient` supports both mTLS-only and mTLS+API-key authentication modes.
**Consequences:** Certificate renewal is an operational concern that must be tracked per state. Failure to renew before expiration will block registrations for that state. A monitoring check validates certificate expiration dates daily.
