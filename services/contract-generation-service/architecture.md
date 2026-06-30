# Contract Generation Service — Architecture Notes

## Component Diagram

```
                         ┌──────────────────────────┐
                         │   contract-generation-    │
                         │        service            │
                         │                           │
  ┌───────────────┐      │  ┌────────────────────┐   │
  │ Kafka Consumer │─────▶│  │ ContractGeneration │   │
  │ (appraisal_   │      │  │     UseCase        │   │
  │  completed)   │      │  └────────┬───────────┘   │
  └───────────────┘      │           │               │
                         │     ┌─────┴─────┐         │
  ┌───────────────┐      │     │           │         │
  │  REST API     │─────▶│  ┌──▼───┐  ┌───▼──────┐  │
  │ (POST/GET)    │      │  │Templ.│  │ Proposal │  │
  └───────────────┘      │  │Engine│  │  Client  │  │
                         │  └──┬───┘  └───┬──────┘  │
                         │     │          │         │
                         └─────┼──────────┼─────────┘
                               │          │
                    ┌──────────▼──┐   ┌───▼──────────┐
                    │ PostgreSQL  │   │  proposal-    │
                    │ contract_db │   │  service      │
                    └─────────────┘   └──────────────┘
                               │
                    ┌──────────▼──┐
                    │   Kafka     │
                    │ (contract_  │
                    │  generated) │
                    └─────────────┘
```

## Data Flow

### Primary: Event-Driven Generation

```
appraisal-report-service
        │
        │  appraisal_completed (Kafka)
        ▼
contract-generation-service
        │
        ├──► GET proposal-service /api/v1/proposals/{id}  (sync REST)
        │
        ├──► SELECT template FROM contract_template WHERE loan_type = ? AND is_active = true
        │
        ├──► Render HTML → PDF (in-process, Thymeleaf + OpenHTMLToPDF)
        │
        ├──► INSERT INTO contract (metadata + PDF reference)
        │
        └──► contract_generated (Kafka)
                │
                ▼
        compliance-check-service
```

### Secondary: REST-Triggered Re-generation

The REST endpoint follows the same internal flow but is triggered by an HTTP POST instead of a Kafka event. This is used by backoffice operators when a contract needs to be regenerated (e.g., after a template fix).

## Integration Patterns

- **Event-driven trigger (Kafka):** The main entry point. Uses at-least-once delivery with idempotency checks based on `proposalId + templateVersion`.
- **Synchronous REST call (proposal-service):** Blocking call with retry (3 attempts, exponential backoff). No circuit breaker — the team accepts the risk because contract generation is not latency-sensitive (it's a background process).
- **Event publishing (Kafka):** After successful generation, publishes `contract_generated` for downstream consumption. Uses transactional outbox pattern to ensure the event is published only if the database commit succeeds.

## ADR-001: Use Thymeleaf + OpenHTMLToPDF for Contract Rendering

**Date:** 2023-06-15
**Status:** Accepted

### Context

We needed a templating engine for generating contract PDFs. Options considered:
1. **JasperReports** — powerful but complex, steep learning curve, XML-based templates
2. **Apache FOP** — XSL-FO based, good for structured docs but poor developer experience
3. **Thymeleaf + OpenHTMLToPDF** — HTML templates rendered to PDF, familiar to web developers

### Decision

We chose Thymeleaf + OpenHTMLToPDF. The legal team already had HTML-based templates from a previous vendor tool, and the web-developer-friendly syntax lowered the barrier for template maintenance.

### Consequences

- **Positive:** Templates are easy to edit and preview in a browser. The legal team can draft templates with minimal developer involvement.
- **Negative:** OpenHTMLToPDF has known memory issues with large documents. We mitigate this with a concurrency semaphore. CSS support is limited compared to modern browsers, so complex layouts require workarounds.

## ADR-002: Transactional Outbox for Event Publishing

**Date:** 2023-07-02
**Status:** Accepted

### Context

We had a reliability issue: if the service crashed after committing the contract to the database but before publishing the Kafka event, downstream services would never know the contract was generated. Conversely, if we published first and the DB commit failed, we'd have a phantom event.

### Decision

Implement the transactional outbox pattern. Contract metadata and the outbox event record are written in the same database transaction. A separate poller reads unpublished outbox records and publishes them to Kafka, marking them as sent.

### Consequences

- **Positive:** Guaranteed consistency between database state and published events. No lost events, no phantom events.
- **Negative:** Slight increase in event delivery latency (poller runs every 500ms). Added complexity in the form of the outbox table and poller component. Acceptable trade-off for a non-latency-sensitive flow.
