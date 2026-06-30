# Eligibility Service

## Overview

The Eligibility Service acts as the first quality gate in the CredityFlow onboarding pipeline. When a new lead comes in (via the `lead_created` event from lead-intake-service), this service checks whether the applicant meets basic eligibility requirements before we invest resources in document collection, credit analysis, and all the heavier downstream processing.

The rules are fairly straightforward today: minimum age (18), geographic coverage (currently all Brazilian states except a few where we do not operate), minimum declared income, and product-specific criteria. But the system is designed to be extensible -- rules are stored in the database and can be toggled on/off or adjusted by the operations team without a deploy.

**Stack:** Kotlin / Spring Boot / PostgreSQL (`eligibility_db`)

## Getting Started

### Prerequisites

- JDK 17+
- Docker / Docker Compose
- Network access to lead-intake-service and customer-service (or use mocks)

### Local Development

```bash
docker-compose up -d postgres kafka
./gradlew bootRun --args='--spring.profiles.active=local'
```

The local profile includes a `DataInitializer` that seeds the database with default eligibility rules on first startup. You will see a log line like `Seeded 12 eligibility rules` if it runs successfully.

To test without real upstream services, enable the mock clients:

```yaml
# application-local.yml
lead-intake-service.mock.enabled: true
customer-service.mock.enabled: true
```

### Tests

```bash
./gradlew test
./gradlew integrationTest
```

The integration tests cover the full flow: publishing a `lead_created` event via an embedded Kafka broker and asserting that a `lead_qualified` event is produced (or not, depending on the test case).

## Architecture

The service is structured in the standard hexagonal style used across CredityFlow:

- **Inbound adapters:** Kafka consumer (for `lead_created`) and REST controller (for on-demand checks)
- **Application layer:** `EligibilityCheckUseCase` orchestrates rule loading, evaluation, and result persistence
- **Domain layer:** `EligibilityRule` and `EligibilityResult` entities, plus the `RuleEvaluator` that runs each rule
- **Outbound adapters:** JPA repositories, Kafka producer, REST clients for lead-intake-service and customer-service

The rule evaluation is simple but effective. Each `EligibilityRule` has a `type` field (AGE_CHECK, INCOME_CHECK, LOCATION_CHECK, PRODUCT_RULE) and a JSON `parameters` column. The `RuleEvaluator` dispatches to the appropriate handler based on the type. Adding a new rule type requires adding a handler class and registering it -- no changes to the core evaluation loop.

One thing worth noting: the `lead_created` event payload intentionally does not contain the full lead data. It includes the lead ID, source channel, and a few basic fields. The eligibility service calls back to `GET /api/v1/leads/{id}` to fetch the complete lead record. This was a deliberate design choice -- see the architecture doc for the reasoning.

## API Reference

### POST /api/v1/eligibility/check

Run an eligibility check on demand. This is used by the customer-portal-bff to give applicants instant feedback before they even submit a full lead.

**Request:**
```json
{
  "leadId": "optional-lead-uuid",
  "age": 25,
  "declaredIncome": 3500.00,
  "state": "SP",
  "productType": "PERSONAL_LOAN"
}
```

**Response (200):**
```json
{
  "eligible": true,
  "rulesEvaluated": 4,
  "rulesPassed": 4,
  "failedRules": [],
  "checkedAt": "2025-06-15T10:30:00Z"
}
```

If the applicant fails a rule, the `failedRules` array includes the rule name and a human-readable reason (e.g., `"Minimum declared income is R$1,500.00"`).

### GET /api/v1/eligibility/rules

Returns all currently active rules. Supports filtering by `productType` query parameter.

## Configuration

| Property | Description | Default |
|---|---|---|
| `spring.datasource.url` | PostgreSQL connection | `jdbc:postgresql://localhost:5432/eligibility_db` |
| `kafka.bootstrap-servers` | Kafka brokers | `localhost:9092` |
| `kafka.topic.lead-created` | Input topic | `onboarding.lead.created` |
| `kafka.topic.lead-qualified` | Output topic | `onboarding.lead.qualified` |
| `kafka.consumer.group-id` | Consumer group | `eligibility-service` |
| `lead-intake-service.base-url` | Lead service URL | `http://lead-intake-service:8080` |
| `customer-service.base-url` | Customer service URL | `http://customer-service:8080` |

## Deployment

Deployed on Kubernetes via Helm. The service runs 2 replicas in production. Since it consumes from a Kafka topic, Kafka handles partition assignment across replicas automatically -- no special coordination needed.

- **Replicas:** 2 (staging and production)
- **Resources:** 512Mi memory, 250m CPU
- **Health:** `/actuator/health`

The Kafka consumer uses auto-commit disabled with manual offset commits after processing. This means if the service crashes mid-check, the event will be re-processed on restart. The eligibility check is idempotent (re-checking a lead just overwrites the previous result), so this is safe.

## Monitoring

### Metrics to Watch

- **`eligibility_checks_total{result=qualified}`** vs **`{result=disqualified}`** -- The ratio tells you how your funnel is performing. Historically about 70% of leads qualify.
- **`rule_failure_total` by rule_name** -- Tells you which rules are filtering out the most people. If `INCOME_CHECK` suddenly starts rejecting 90% of leads, someone probably changed the threshold.
- **Consumer lag on `onboarding.lead.created`** -- Should stay close to zero. If it climbs, the service is not keeping up with lead volume.

### Alerts

- Disqualification rate above 95% for 10+ minutes -- Something is probably wrong with a rule configuration.
- Consumer lag > 1000 -- Service is falling behind.

## Known Issues

1. **Rule ordering matters but is not enforced.** Rules are evaluated in database insertion order. There is a `priority` column but it is not consistently populated. Fixing this is on the backlog.

2. **No dry-run mode for rule changes.** When ops changes a rule parameter, there is no way to preview the impact on recent leads. The team has discussed building a shadow-evaluation mode but it has not been started.

3. **Customer-service dependency can slow things down.** The call to customer-service to check for returning applicants adds latency. If customer-service is slow or down, the eligibility check falls back to treating the lead as new, which is safe but means we might miss some returning customer optimizations.
