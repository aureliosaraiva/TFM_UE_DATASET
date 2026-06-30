# Architecture: registry-tracker-service

## Component Diagram

```
  +-------------------------+       +---------------------------+
  | detran-integration-svc  |       | cartorio-integration-svc  |
  +------------+------------+       +-------------+-------------+
               |                                  |
  detran_lien_registered (Kafka)    cartorio_lien_registered (Kafka)
               |                                  |
               +----------------+-----------------+
                                |
                                v
+-----------------------------------------------------------------------+
|                    registry-tracker-service                           |
|                                                                       |
|  +----------------+    +------------------------------+               |
|  | EventConsumer  |--->| RegistrationTrackingService   |              |
|  | (Kafka)        |    +---+---------+------------+---+               |
|  +----------------+        |         |            |                   |
|                            v         v            v                   |
|  +--------------------+  +--------+ +----------+ +----------------+  |
|  | TrackingController |  | Retry  | | Status   | | EventPublisher |  |
|  | (REST API)         |  | Sched. | | Aggreg.  | | (Kafka)        |  |
|  +--------------------+  +--------+ +----------+ +----------------+  |
|                                                                       |
|                          +------------------+                         |
|                          | PostgreSQL       |                         |
|                          | registry_db      |                         |
|                          +------------------+                         |
+-----------------------------------------------------------------------+
                                |
               +----------------+----------------+
               |                                 |
  registry_completed (Kafka)        registry_failed (Kafka)
               |                                 |
               v                                 v
  +---------------------+          +--------------------+
  | disbursement-service |          | (ops notification) |
  +---------------------+          +--------------------+
```

## Data Flow

1. **Status Events Received** -- The service consumes `detran_lien_registered` and `cartorio_lien_registered` events from Kafka.
2. **Status Aggregation** -- Each event is mapped to a `RegistrationStatus` record. If the event indicates success, a `CompletionConfirmation` is created. If failure, a `RetryRecord` is created.
3. **Retry Logic** -- The `RetryScheduler` runs every 30 seconds, picking up retries that are due. Backoff follows: 60s, 120s, 240s, 480s, 960s (capped at 3600s). A retry re-publishes the original `registry_requested` event to trigger the integration service again.
4. **Completion** -- On successful registration (or successful retry), a `registry_completed` event is published. This unblocks the disbursement pipeline.
5. **Permanent Failure** -- After 5 failed attempts, a `registry_failed` event is published and the operations team is notified for manual resolution.

## Integrations

| System | Direction | Protocol | Purpose |
|---|---|---|---|
| detran-integration-service | Inbound | Kafka event | Receives DETRAN registration results |
| cartorio-integration-service | Inbound | Kafka event | Receives cartorio registration results |
| disbursement-service | Outbound | Kafka event | Publishes registry_completed to trigger disbursement |
| registry-orchestrator-service | Outbound | Kafka event | Re-publishes registry_requested for retries |
| PostgreSQL (registry_db) | Internal | JDBC | Persists tracking data and retry records |

## Architecture Decision Records

### ADR-034-1: Exponential Backoff with Jitter for Retry Logic

**Status:** Accepted
**Context:** Registration failures with external systems (DETRAN, cartorio) are often transient -- network timeouts, rate limits, or temporary unavailability. Immediate retries can exacerbate the problem, while fixed delays are inefficient.
**Decision:** Implement exponential backoff with jitter for retry scheduling. The initial delay is 60 seconds, multiplied by 2.0 for each subsequent attempt, capped at 3600 seconds. A random jitter of +/-20% is applied to prevent thundering herd effects when multiple registrations fail simultaneously.
**Consequences:** Retries are spread over time, reducing load on external systems during outages. The maximum retry window (5 attempts) spans approximately 30 minutes, which is acceptable for the registration timeline. Persistent failures (beyond 5 attempts) require manual intervention.

### ADR-034-2: Unified Tracking Model Across Registration Channels

**Status:** Accepted
**Context:** DETRAN and cartorio registrations have different processing characteristics (near-instant vs. multi-day) but downstream consumers (disbursement-service) need a single, channel-agnostic completion signal.
**Decision:** The tracker maintains a unified `RegistrationStatus` model that abstracts the channel-specific details. Both `detran_lien_registered` and `cartorio_lien_registered` events are mapped to the same status lifecycle. The `registry_completed` event carries no channel-specific information, allowing downstream services to remain decoupled from the registration channel.
**Consequences:** Channel-specific debugging requires looking at the underlying integration service logs. The tracker provides a high-level view suitable for pipeline progression and operational monitoring.
