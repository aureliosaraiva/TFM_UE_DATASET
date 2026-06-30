# Architecture: cartorio-integration-service

## Component Diagram

```
                     +----------------------------+
                     | registry-orchestrator-svc  |
                     +-------------+--------------+
                                   |
                      registry_requested (Kafka)
                        [channel=CARTORIO]
                                   |
                                   v
+------------------------------------------------------------------------+
|                   cartorio-integration-service                         |
|                                                                        |
|  +----------------+    +-----------------------------+                 |
|  | EventConsumer  |--->| CartorioRegistrationService  |                |
|  | (Kafka)        |    +---+---------------------+---+                 |
|  +----------------+        |                     |                     |
|                            v                     v                     |
|  +--------------------+   +---------------------+   +----------------+ |
|  | CartorioController |   | StatusPollingScheduler|  | EventPublisher | |
|  | (REST API)         |   | (every 4 hours)      |  | (Kafka)        | |
|  +--------------------+   +----------+----------+   +----------------+ |
|                                      |                                 |
|                           +----------v-----------+                     |
|                           | CartorioDigitalClient |                    |
|                           | (OAuth2 HTTP client)  |                    |
|                           +----------+-----------+                     |
|                                      |           +------------------+  |
|                                      |           | PostgreSQL       |  |
|                                      |           | registry_db      |  |
|                                      |           +------------------+  |
+------------------------------------------------------------------------+
                                       |
                                       v (HTTPS + OAuth2)
                          +-------------------------+
                          |    Cartorio Digital      |
                          |    (National Platform)   |
                          +-------------------------+

                     cartorio_lien_registered (Kafka)
                                       |
                                       v
                     +----------------------------+
                     | registry-tracker-service   |
                     +----------------------------+
```

## Data Flow

1. **Event Received** -- The service consumes `registry_requested` events from Kafka, filtered by `channel=CARTORIO` header.
2. **Submission** -- The `CartorioDigitalClient` submits the lien registration request via the Cartorio Digital REST API using OAuth2 client credentials authentication.
3. **Asynchronous Processing** -- The cartorio processes the registration offline. This takes 3-15 business days. The service persists the registration with status `SUBMITTED`.
4. **Status Polling** -- The `StatusPollingScheduler` runs every 4 hours, querying Cartorio Digital for updates on all pending registrations. Status transitions are recorded in PostgreSQL.
5. **Exigencia Handling** -- If the cartorio raises an exigencia (request for additional information), the registration status changes to `EXIGENCIA` and a notification is sent to the operations team.
6. **Completion** -- When the cartorio confirms registration on the matricula, a `cartorio_lien_registered` event is published to Kafka.

## Integrations

| System | Direction | Protocol | Purpose |
|---|---|---|---|
| registry-orchestrator-service | Inbound | Kafka event | Receives registry_requested for property liens |
| Cartorio Digital | Outbound | REST + OAuth2 | Submits and queries property lien registrations |
| registry-tracker-service | Outbound | Kafka event | Publishes cartorio_lien_registered for tracking |
| PostgreSQL (registry_db) | Internal | JDBC | Persists registration data and property info |

## Architecture Decision Records

### ADR-033-1: Polling-Based Status Updates from Cartorio Digital

**Status:** Accepted
**Context:** Cartorio Digital does not provide real-time webhooks for registration status changes. The platform offers a status query API that returns the current state of a registration.
**Decision:** Implement a scheduled polling job that runs every 4 hours to check the status of all pending registrations. The interval balances API rate limits against timely status updates. Polling stops after 30 days for registrations that have not completed (considered stale).
**Consequences:** Status updates may be delayed by up to 4 hours from the actual state change at the cartorio. This is acceptable given the multi-day processing timeline. The polling job must handle pagination for high volumes of pending registrations.

### ADR-033-2: Explicit Exigencia State in Domain Model

**Status:** Accepted
**Context:** Cartorios frequently raise "exigencias" -- requests for corrections or additional documentation before completing a registration. This is a normal part of the process that occurs in roughly 15-20% of registrations.
**Decision:** Model EXIGENCIA as a first-class status in the CartorioRegistration lifecycle. When detected during polling, the service records the exigencia details and triggers a notification to the operations team. The registration remains in EXIGENCIA until manually resolved and resubmitted.
**Consequences:** Exigencia resolution is a manual process that cannot be fully automated. The operations team must monitor the exigencia alert and respond within the SLA. Unresolved exigencias block the lending pipeline for the affected contract.
