# Appraisal Orchestrator Service (svc-23)

**Domain:** Appraisal
**Stack:** Kotlin / Spring Boot
**Database:** PostgreSQL (`appraisal_db`)
**Deployment:** AWS EKS

## Component Overview

Secured lending requires collateral, and collateral requires appraisal. The Appraisal Orchestrator Service manages the process of determining the value of the asset a customer pledges against their loan. CredityFlow supports two collateral types -- vehicles and real estate properties -- each with fundamentally different appraisal workflows, data sources, and turnaround times.

Rather than building separate appraisal services for each collateral type, CredityFlow uses this orchestrator to provide a unified interface to the rest of the platform while routing internally to the appropriate appraisal sub-flow. To the Proposal Service, there is only one appraisal event to wait for; the orchestrator handles the complexity underneath.

### Components

**Collateral Router:** Inspects the proposal's collateral type (received in the triggering event or fetched from the Proposal Service) and routes to the correct appraisal workflow. Currently two paths exist:

- **Vehicle Appraisal Workflow:** Queries the FIPE table (the Brazilian standard for vehicle pricing) via an external API, applies depreciation adjustments based on mileage and condition (reported by the customer and verified through photos), and produces a valuation. This is largely automated, completing in seconds.

- **Property Appraisal Workflow:** Initiates a request to a partner appraisal company (via API integration) for an in-person property inspection and valuation report. This is a long-running process -- typically 5-15 business days. The orchestrator tracks the request status by polling the partner's API periodically (via a scheduled job) and processing webhook callbacks when available.

**Valuation Adjuster:** Applies CredityFlow's internal haircut rules to the raw appraisal value. For example, vehicle valuations receive a 10% haircut (forced sale discount); property valuations receive a 15-20% haircut depending on property type and location. These haircuts produce the "collateral value for lending purposes," which determines the maximum LTV.

**Status Tracker:** Maintains the state of each appraisal request in PostgreSQL: `INITIATED`, `IN_PROGRESS`, `VALUATION_RECEIVED`, `ADJUSTED`, `COMPLETED`, `FAILED`. For property appraisals, intermediate status updates are stored as they arrive from the partner.

**Event Publisher:** Emits `AppraisalCompleted` (with the adjusted collateral value) or `AppraisalFailed` events to SNS when the process concludes.

## Data Flow

**Vehicle appraisal (fast path):**
1. `CreditDecisionRendered` event with `decision=APPROVED` triggers the orchestrator.
2. Collateral Router identifies the collateral as a vehicle.
3. FIPE API is called with the vehicle's make, model, year, and variant.
4. Depreciation adjustments are applied based on reported mileage and condition.
5. Valuation Adjuster applies the forced sale haircut.
6. Result is persisted and `AppraisalCompleted` is published. Total time: 2-5 seconds.

**Property appraisal (slow path):**
1. Same trigger as above, but collateral is identified as real estate.
2. An appraisal request is submitted to the partner company via their API.
3. The orchestrator records the request and enters a waiting state.
4. A scheduled job polls the partner API every 4 hours for status updates. Webhook callbacks provide real-time updates when available.
5. When the partner's valuation report arrives, it is stored and the Valuation Adjuster applies the haircut.
6. `AppraisalCompleted` is published. Total time: 5-15 business days.

## Integration Patterns

- **Inbound (async):** SQS subscription to `CreditDecisionRendered` events (filtered for approved decisions).
- **Outbound (async):** `AppraisalCompleted` / `AppraisalFailed` events to SNS, consumed by `proposal-service`.
- **External (sync):** FIPE API for vehicle pricing. Partner appraisal company API for property valuation requests and status checks.
- **Webhook (inbound):** REST endpoint for partner appraisal company callbacks.
- **Scheduled:** Cron job (Kubernetes CronJob) for polling partner API on pending property appraisals.

## ADR-001: Unified Orchestrator vs. Separate Vehicle and Property Services

**Context:** Vehicle and property appraisals are different enough that separate services seemed natural. However, from the perspective of the rest of the platform, "appraisal" is a single step in the proposal lifecycle. Two separate services would mean two event types, conditional subscription logic in the Proposal Service, and duplicated haircut/adjustment logic.

**Decision:** Build a single orchestrator that presents a unified interface (`AppraisalCompleted` event) while routing internally to collateral-type-specific workflows. If a third collateral type is added in the future (e.g., equipment), a new internal workflow is added without changing the external contract.

**Consequences:**
- Clean integration surface: the Proposal Service subscribes to one event type regardless of collateral type.
- Internal complexity is higher than two focused services, but contained within well-separated workflow classes.
- Risk: if the service grows to support many collateral types, it could become a monolith. The team has agreed to revisit this if a fourth collateral type is introduced.

## ADR-002: Hybrid Polling + Webhook for Property Appraisal Status

**Context:** The partner appraisal company offers two mechanisms for status updates: a GET endpoint for polling and optional webhook callbacks. Webhooks are faster but unreliable -- the partner's webhook delivery has a ~90% success rate based on historical data. Polling alone would introduce unnecessary latency (up to 4 hours between checks).

**Decision:** Use both. Register a webhook endpoint for real-time updates. Run a polling job every 4 hours as a fallback to catch any missed webhooks. Deduplicate status updates in the database (idempotent status transitions) so that receiving the same update from both sources does not cause issues.

**Consequences:**
- Average notification latency is near-real-time (via webhook) with guaranteed eventual consistency (via polling).
- The 4-hour polling interval keeps API call volume low (partner charges per API call above a monthly threshold).
- Idempotent status handling adds minor code complexity but prevents state corruption from duplicate updates.
