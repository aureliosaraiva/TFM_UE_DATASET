# Appraisal Orchestrator Service

```
Service: svc-23 | Domain: Appraisal | Team: team-appraisal | Stack: Kotlin/Spring Boot
```

## Overview

When a loan is approved, the collateral needs to be valued. But what *kind* of collateral is it? A 2019 Honda Civic or a two-bedroom flat in Sao Paulo? The answer determines which appraisal process to follow — and that routing decision is exactly what this service handles.

The Appraisal Orchestrator is the single entry point into the appraisal domain. It listens for `credit_approved` events, inspects the collateral type attached to the proposal, and dispatches the request to either `vehicle-appraisal-service` or `property-appraisal-service`. It also tracks the lifecycle of each appraisal request, handling timeouts and providing status queries.

Think of it as a traffic controller: it does not perform valuations itself, but it ensures every approved loan reaches the right valuation specialist.

## Getting Started

```bash
# Infrastructure
docker-compose up -d postgres kafka

# Environment
export DATABASE_URL=jdbc:postgresql://localhost:5432/appraisal_db
export DATABASE_USERNAME=appraisal_svc
export DATABASE_PASSWORD=changeme
export KAFKA_BOOTSTRAP_SERVERS=localhost:9092
export PROPOSAL_SERVICE_URL=http://localhost:8081

# Run
./gradlew bootRun

# Tests
./gradlew test
./gradlew integrationTest
```

Health endpoint: `GET /actuator/health` on port 8080.

## Architecture

The orchestrator follows a **saga-like pattern** without a full saga framework. It creates an `AppraisalRequest` record, publishes a routed event, and then monitors for completion events from the downstream services. If no completion arrives within the configured timeout, it marks the request as TIMED_OUT and alerts the operations team.

```
credit_approved
      |
      v
[appraisal-orchestrator-service]
      |
      +---> collateral_type = VEHICLE ---> appraisal_requested (vehicle topic)
      |
      +---> collateral_type = PROPERTY --> appraisal_requested (property topic)
```

State machine for `AppraisalStatus`:
```
REQUESTED --> IN_PROGRESS --> COMPLETED
                          --> FAILED
         --> TIMED_OUT
```

Full details in [architecture.md](./architecture.md).

## API Reference

### POST /api/v1/appraisals

Creates a new appraisal request manually (typically used by backoffice).

**Request:**
```json
{
  "proposal_id": "uuid",
  "collateral_type": "VEHICLE",
  "collateral_details": {
    "make": "Honda",
    "model": "Civic",
    "year": 2019,
    "plate": "ABC1234"
  }
}
```

### GET /api/v1/appraisals/{id}

Returns appraisal request status, collateral type, timestamps, and linked valuation reference.

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection | — |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka brokers | `localhost:9092` |
| `PROPOSAL_SERVICE_URL` | URL for proposal-service | — |
| `APPRAISAL_TIMEOUT_VEHICLE_HOURS` | Timeout for vehicle appraisals | `24` |
| `APPRAISAL_TIMEOUT_PROPERTY_HOURS` | Timeout for property appraisals | `168` |

## Deployment

```bash
helm upgrade --install appraisal-orchestrator ./charts/appraisal-orchestrator-service \
  --namespace appraisal \
  --values values-production.yaml
```

Lightweight service: 256Mi memory, 128m CPU request. 2 replicas minimum. Kafka partition count should match replica count.

## Monitoring

- **Dashboard:** "Appraisal Pipeline Overview" — request volume by collateral type, status funnel, completion times
- **Dashboard:** "Routing Performance" — routing latency, error rates, timeout trends

**Alerts:**
- `appraisal_routing_failure` — routing errors > 2% over 5 min
- `appraisal_timeout_spike` — timeout rate increases 50% in 1 hour

## Known Issues

1. **Two collateral types only** — The routing logic is a simple if/else on `CollateralType`. Adding a new type (e.g., machinery, accounts receivable) requires a code change and redeployment. A plugin-based router is on the roadmap for when the third collateral type is introduced.

2. **Proposal-service dependency** — The orchestrator fetches collateral details from `proposal-service` synchronously during routing. If proposal-service is down, routing fails even though the `credit_approved` event has already been consumed. The event is retried, but prolonged outages can cause consumer lag to build up.

3. **Timeout granularity** — Timeouts are checked by a scheduled job that runs every 15 minutes. This means a timed-out appraisal may not be detected for up to 15 minutes after the actual timeout. Acceptable for current SLAs but may need improvement.
