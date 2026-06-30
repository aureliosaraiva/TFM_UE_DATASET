# Eligibility Service — Architecture Notes

## Component Diagram

```
[Kafka: onboarding.lead.created]
    |
    v
[LeadCreatedConsumer]
    |
    v
[EligibilityCheckUseCase]
    |
    +---> [LeadIntakeClient] --HTTP--> [lead-intake-service]
    +---> [CustomerServiceClient] --HTTP--> [customer-service]
    +---> [RuleRepository] --JPA--> [eligibility_db]
    +---> [ResultRepository] --JPA--> [eligibility_db]
    |
    v
[KafkaProducer] --> [Kafka: onboarding.lead.qualified]

[REST: EligibilityController]
    |
    v
[EligibilityCheckUseCase] (same as above)
```

The service has two entry points: the Kafka consumer for event-driven checks and the REST controller for on-demand checks. Both converge on the same `EligibilityCheckUseCase`.

## Data Flow

1. The `LeadCreatedConsumer` receives a `lead_created` event containing the lead ID and minimal metadata.
2. `EligibilityCheckUseCase` is invoked. It first calls `lead-intake-service` (GET /api/v1/leads/{id}) to fetch the full lead record. This synchronous call is necessary because the event is intentionally kept small.
3. Optionally, the use case calls `customer-service` to check if this CPF matches an existing customer (returning applicants may get fast-tracked in future iterations, though today it is purely informational).
4. Active `EligibilityRule` records are loaded from the database.
5. The `RuleEvaluator` iterates over the rules, dispatching to typed handlers (AgeCheckHandler, IncomeCheckHandler, LocationCheckHandler, ProductRuleHandler).
6. An `EligibilityResult` is persisted with pass/fail status and details of each rule evaluation.
7. If the lead qualifies, a `lead_qualified` event is published to Kafka.

## Integration Patterns

- **Event consumption:** Standard Kafka consumer with manual offset management. Consumer group is `eligibility-service`. The topic has 4 partitions, and with 2 service replicas, each instance handles 2 partitions.
- **Synchronous REST calls:** Uses Spring WebClient with Resilience4j circuit breakers for both lead-intake-service and customer-service. Timeout is 3 seconds. If lead-intake-service is unreachable, the check fails (we cannot evaluate without lead data). If customer-service is unreachable, we proceed without the returning-customer check.
- **Event publishing:** Direct Kafka publishing (no outbox pattern here -- if publishing fails, the consumer will retry the entire message since the offset is not committed).

## Architectural Decision Records

### ADR-001: Lean Event Payloads with Callback

**Context:**
When designing the `lead_created` event schema, we debated between including the full lead data in the event ("fat event") versus including only the ID and requiring consumers to call back for details ("thin event").

**Decision:**
Use thin events. The `lead_created` event contains the lead ID, source channel, and timestamp. Consumers that need more data must call `GET /api/v1/leads/{id}`.

**Consequences:**
- Positive: Events are small, fast to serialize, and do not contain PII. This simplifies event storage, logging, and compliance.
- Positive: If the lead data schema changes, events do not need to be versioned as aggressively.
- Negative: Every consumer that needs lead details makes an additional REST call, adding latency and coupling.
- Negative: If lead-intake-service is down, consumers that need the full record cannot process events.
- Trade-off: This works well for our current scale (hundreds of leads per hour). At significantly higher volumes, we might revisit.

### ADR-002: Database-Driven Rule Configuration

**Context:**
Eligibility rules change fairly often -- the business team adjusts income thresholds, adds new geographic restrictions, or tweaks product requirements. Initially, rules were hardcoded in the application. Every change required a code change and deploy.

**Decision:**
Store rules in the database with a typed handler pattern. Each rule has a `type` (enum), `parameters` (JSON column), `enabled` flag, and optional effective date range. The application loads rules at check time (with a short cache, TTL 60 seconds).

**Consequences:**
- Positive: Business and ops teams can update rules via a simple database update or admin API without deploying code.
- Positive: Rules can be enabled/disabled and scheduled with effective date ranges.
- Positive: Historical rule configurations are preserved for audit.
- Negative: Adding an entirely new rule *type* still requires a code change (new handler class).
- Negative: The JSON parameters column is flexible but not type-safe. Misconfigured parameters can cause runtime errors. We mitigate this with validation on rule creation/update.
- Negative: The 60-second cache means rule changes are not instant. In practice this has not been a problem.
