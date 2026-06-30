# Vehicle Appraisal Service (svc-24)

**Domain:** Appraisal | **Stack:** Kotlin / Spring Boot | **Database:** PostgreSQL | **Status:** Production

## Component Overview

The Vehicle Appraisal Service is responsible for determining the market value of vehicles offered as collateral in secured lending operations. It acts as the system of record for all vehicle valuations within the CredityFlow platform.

### Core Responsibilities

- Accept appraisal requests for vehicles identified by plate, chassis, or Renavam number.
- Query the FIPE-API (Tabela FIPE) for reference pricing based on make, model, year, and fuel type.
- Apply internal depreciation curves and condition adjustments to derive a final appraised value.
- Store appraisal history for audit and recalculation purposes.
- Publish valuation results so downstream services (credit decisioning, collateral management) can react.

### Key Components

| Component | Purpose |
|---|---|
| `AppraisalController` | REST endpoints for creating and querying appraisals |
| `FipeIntegrationClient` | Resilience4j-wrapped HTTP client calling FIPE-API |
| `ValuationEngine` | Business logic: adjustments, depreciation, mileage factors |
| `AppraisalRepository` | JPA repository against `appraisal_db.vehicle_appraisals` |
| `AppraisalEventPublisher` | Publishes `VehicleAppraised` events to SNS |

## Data Flow

1. The origination workflow (or backoffice BFF) sends a `POST /v1/appraisals` request with vehicle identifiers and condition metadata.
2. The service resolves the FIPE reference code via a lookup cache (Redis, TTL 24h) or falls back to the FIPE-API.
3. The `ValuationEngine` applies business rules: mileage penalty, condition grade multiplier, regional adjustment.
4. The final appraisal record is persisted in PostgreSQL with status `COMPLETED` or `FAILED`.
5. A `VehicleAppraised` event is published to `arn:aws:sns:us-east-1:*:appraisal-events` for consumption by the credit decision pipeline and the appraisal report service.

```
Origination --> [POST /v1/appraisals] --> VehicleAppraisalService
                                              |
                                              +--> FIPE-API (external)
                                              |
                                              +--> PostgreSQL (appraisal_db)
                                              |
                                              +--> SNS (appraisal-events)
```

## Integration Patterns

- **FIPE-API:** Synchronous REST call protected by a circuit breaker (half-open after 30s, failure threshold 5). Responses are cached in Redis to reduce external call volume and protect against FIPE downtime.
- **SNS/SQS:** The service publishes domain events; it does not directly call other CredityFlow services. This keeps coupling loose and lets consumers (e.g., `appraisal-report-service`, `case-management-service`) evolve independently.
- **Idempotency:** Appraisal requests carry a client-generated idempotency key stored in a unique index. Duplicate submissions return the existing result.

## Architecture Decision Records

### ADR-024-001: Use FIPE-API as the single external pricing source

**Context:** Multiple vehicle pricing providers exist in the Brazilian market (FIPE, KBB Brasil, Molicar). The team evaluated accuracy, cost, and API reliability.

**Decision:** Adopt FIPE-API as the sole pricing source, with an aggressive caching layer to mitigate its occasional instability.

**Consequences:**
- Simpler integration surface; one adapter to maintain.
- Risk of single-provider dependency mitigated by 24-hour cache and a manual override endpoint for operations.
- If FIPE changes their API contract, only this service is affected.

### ADR-024-002: Store raw FIPE response alongside computed valuation

**Context:** Auditors and compliance reviewers need to trace how a final valuation was derived. Storing only the computed value would lose the input data.

**Decision:** Persist the full FIPE JSON payload in a `jsonb` column (`raw_fipe_response`) alongside the calculated fields.

**Consequences:**
- Increases row size but enables full traceability without replaying external calls.
- Enables offline analysis of FIPE price drift over time.
- Data retention policy must account for the larger storage footprint (currently 2-year retention).
