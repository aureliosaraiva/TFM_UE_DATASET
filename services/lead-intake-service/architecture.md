# Lead Intake Service — Architecture Notes

## Component Diagram

The service is composed of three main layers:

```
[External Clients]
    |
    v
[API Layer: LeadController, PartnerLeadController]
    |
    v
[Domain Layer: LeadService, DuplicateChecker, PartnerAuthClient]
    |
    +---> [Infrastructure: LeadRepository (JPA/PostgreSQL)]
    +---> [Infrastructure: OutboxRepository (JPA/PostgreSQL)]
    +---> [Infrastructure: AuthServiceClient (HTTP/REST)]
    +---> [Infrastructure: OutboxPoller -> KafkaProducer]
```

External clients include the CredityFlow website (via customer-portal-bff), the mobile app (direct), and partner systems (via PartnerLeadController with auth-service token validation).

## Data Flow

1. A lead submission arrives via HTTP POST.
2. If the request originates from a partner channel, the `Authorization` header is forwarded to auth-service for validation. This is a synchronous call — if auth-service is down, partner submissions fail (website and mobile submissions are unaffected since they use session-based auth handled upstream).
3. The `DuplicateChecker` queries recent leads (configurable window, default 72 hours) by CPF and email.
4. The lead is persisted in a single transaction that also writes an outbox record.
5. The `OutboxPoller` (a Spring `@Scheduled` task) reads unpublished outbox entries and pushes them to Kafka as `lead_created` events.
6. Once Kafka acknowledges, the outbox entry is marked as published.

Downstream, the eligibility-service consumes `lead_created` events and kicks off eligibility checks. The customer-portal-bff also reads lead data via the GET endpoints to show submission status to the user.

## Integration Patterns

- **Synchronous:** REST calls to auth-service for partner authentication. Uses Resilience4j circuit breaker with a 3-second timeout and 5-call failure threshold.
- **Asynchronous:** Kafka-based event publishing via transactional outbox. Topic: `onboarding.lead.created`. The event payload includes the lead ID, source channel, and a minimal set of contact details (enough for eligibility-service to start, but not the full PII payload — consumers must call back to GET /api/v1/leads/{id} if they need everything).

## Architectural Decision Records

### ADR-001: Transactional Outbox over Direct Kafka Publishing

**Context:**
Early versions of this service published directly to Kafka inside the request handler. This led to inconsistencies when the database write succeeded but Kafka publishing failed (or vice versa). We saw occasional "phantom leads" — leads that existed in the database but never triggered eligibility checks because the event was lost.

**Decision:**
Adopt the transactional outbox pattern. The lead record and an outbox event record are written in a single database transaction. A separate poller reads the outbox and publishes to Kafka, retrying on failure.

**Consequences:**
- Positive: Guaranteed at-least-once event delivery. No more phantom leads.
- Positive: The service can tolerate temporary Kafka outages without losing events.
- Negative: Introduces a small delay (up to the poll interval, default 5 seconds) between lead creation and event publishing.
- Negative: The outbox table needs periodic cleanup (we run a nightly job to delete entries older than 7 days).
- Negative: Only one instance should poll at a time, so we use a PostgreSQL advisory lock, which adds minor complexity.

### ADR-002: Separate Partner API Controller

**Context:**
Initially, partner submissions and direct submissions shared the same endpoint. This made it difficult to apply partner-specific logic (authentication, rate limiting, field mapping) without cluttering the main controller.

**Decision:**
Create a dedicated `PartnerLeadController` that handles partner-specific concerns (auth-service token validation, partner channel lookup, rate limiting) before delegating to the shared `LeadService`.

**Consequences:**
- Positive: Clean separation of concerns. Partner-specific logic is isolated.
- Positive: Rate limiting can be configured per partner channel.
- Positive: Easier to add new partner integrations without touching the core lead creation path.
- Negative: Some code duplication in request validation (mitigated by shared validation utility).
- Negative: Two controllers to maintain instead of one, though the partner controller is relatively thin.
