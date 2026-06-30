# Property Appraisal Service

> **Service ID:** svc-25 | **Domain:** Appraisal | **Runtime:** Kotlin on Spring Boot 3.x | **Persistence:** PostgreSQL

## Overview

This service handles the valuation of real estate assets used as collateral in CredityFlow's secured lending products. Unlike vehicle appraisals, property valuations require physical or remote inspections. The Property Appraisal Service orchestrates the inspection lifecycle by integrating with InspecaoExpress, a third-party inspection provider, and stores the resulting valuation alongside supporting evidence (photos, geo-coordinates, comparables).

The service operates as an asynchronous workflow: an appraisal is requested, an inspection is dispatched, and the final valuation is computed only after the inspection report is received via webhook.

## Component Breakdown

- **AppraisalRequestHandler** -- Receives origination or backoffice requests. Validates the property address against the address normalization service and creates an appraisal record in `PENDING_INSPECTION` state.
- **InspecaoExpressAdapter** -- Outbound integration. Submits inspection orders to InspecaoExpress's REST API and maps their proprietary status codes to internal domain events.
- **InspectionWebhookController** -- Inbound webhook endpoint. Receives callbacks from InspecaoExpress when an inspection is completed, cancelled, or rescheduled.
- **PropertyValuationEngine** -- Takes the inspection report (condition score, area measurements, comparable sales) and computes the appraised value using a weighted average model.
- **EventPublisher** -- Emits `PropertyAppraised` or `PropertyAppraisalFailed` to the appraisal SNS topic.

## Data Flow

```
                     +------------------+
  Backoffice BFF --> | POST /appraisals | --> appraisal_db (status: PENDING_INSPECTION)
                     +------------------+
                              |
                              v
                     +--------------------+
                     | InspecaoExpress API |  (dispatch inspection order)
                     +--------------------+
                              |
                        (async, hours/days)
                              |
                              v
                     +---------------------+
                     | Webhook /callback   | --> appraisal_db (status: INSPECTED)
                     +---------------------+
                              |
                              v
                     +------------------------+
                     | PropertyValuationEngine | --> appraisal_db (status: COMPLETED)
                     +------------------------+
                              |
                              v
                     SNS: appraisal-events (PropertyAppraised)
```

The asynchronous nature of inspections means appraisals can remain in `PENDING_INSPECTION` for days. A scheduled job runs every 6 hours to detect stale inspections (>72h) and escalates them to the case management service.

## Integration Patterns

**InspecaoExpress (outbound):** Synchronous REST POST to create orders; responses confirmed via async webhook. A shared HMAC secret validates webhook authenticity. Retry with exponential backoff on order creation failures.

**Event-driven downstream:** Property appraisal results are consumed by `appraisal-report-service` (to generate PDF) and the credit decisioning pipeline (to update LTV calculations). Both subscribe to the `appraisal-events` SNS topic through dedicated SQS queues.

**Idempotency:** Webhook deliveries from InspecaoExpress may be duplicated. The service uses the external `inspection_id` as a natural idempotency key, ignoring callbacks for already-completed inspections.

## ADR Notes

### ADR-025-001: Webhook-based inspection completion over polling

**Context:** InspecaoExpress supports both polling (GET status) and webhooks. Inspections take hours to days, making polling wasteful and latency-prone.

**Decision:** Use webhook-based notification for inspection completion. Implement a staleness detector as a safety net for missed webhooks.

**Consequences:** Reduces API call volume by ~95% compared to polling. Introduces a dependency on InspecaoExpress's webhook reliability, mitigated by the 6-hour staleness cron. Requires a publicly reachable webhook endpoint secured with HMAC validation.

### ADR-025-002: Weighted average model for property valuation

**Context:** The team considered adopting a machine-learning model for property valuation but lacked sufficient training data in the early product phase.

**Decision:** Use a deterministic weighted average model (50% comparable sales, 30% cost approach, 20% income approach) that is transparent and auditable.

**Consequences:** Easier to explain valuations to regulators and borrowers. The model can be replaced with an ML-based approach once enough historical data is collected. Manual override capability preserved for edge cases.
