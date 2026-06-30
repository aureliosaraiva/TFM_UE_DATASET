# Benchmark Report — unstructured

**Run ID:** `unstructured__claude-sonnet-4-6__r3__20260623T180415-349199`  
**Model:** `claude-sonnet-4-6`  
**Total questions:** 120

## Summary

| Metric | Value |
|--------|-------|
| Correct | 29 |
| Partial | 74 |
| Incorrect | 15 |
| Errors | 2 |
| Success rate | 55.0% |
| Avg latency | 10719 ms |
| Total cost | $0.5367 |

## LLM-as-Judge

| Dimension | Mean |
|-----------|------|
| correctness | 2.061 |
| completeness | 1.486 |
| groundedness | 2.883 |
| clarity | 2.950 |
| average | 2.345 |
| n (judged) | 120 |

### Judge — per category

| Category | n | correctness | completeness | groundedness | clarity | average |
|---|---|---|---|---|---|---|
| ambiguous | 15 | 2.133 | 1.356 | 2.933 | 3.000 | 2.356 |
| discovery | 18 | 1.926 | 1.315 | 2.944 | 3.000 | 2.296 |
| explanatory | 22 | 2.303 | 1.788 | 2.849 | 2.864 | 2.451 |
| factual | 24 | 2.250 | 1.931 | 2.972 | 3.000 | 2.538 |
| impact | 21 | 1.905 | 1.190 | 2.809 | 3.000 | 2.226 |
| relational | 20 | 1.800 | 1.183 | 2.800 | 2.850 | 2.158 |

## Results by Category

| Category | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| ambiguous | 7 | 8 | 0 | 0 | 15 |
| discovery | 3 | 14 | 1 | 0 | 18 |
| explanatory | 6 | 14 | 1 | 1 | 22 |
| factual | 3 | 11 | 10 | 0 | 24 |
| impact | 7 | 14 | 0 | 0 | 21 |
| relational | 3 | 13 | 3 | 1 | 20 |

## Results by Difficulty

| Difficulty | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| easy | 8 | 20 | 8 | 0 | 36 |
| hard | 9 | 31 | 3 | 1 | 44 |
| medium | 12 | 23 | 4 | 1 | 40 |

## Results by Knowledge Type

| Required Knowledge Type | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| architectural | 5 | 26 | 3 | 1 | 35 |
| factual | 4 | 13 | 9 | 0 | 26 |
| mixed | 10 | 14 | 1 | 0 | 25 |
| operational | 6 | 11 | 0 | 0 | 17 |
| relational | 4 | 10 | 2 | 1 | 17 |

## Correct Answer Examples

### `Q001`
**Q:** Which team owns the lead-intake-service?
**Expected:** The lead-intake-service is owned by team-onboarding. This team is responsible for the initial customer acquisition flow, including lead capture, eligibility checks, and proposal creation.
**Actual:** The provided documentation does not contain any information about team ownership of the Lead Intake Service. The docs cover its architecture, tech stack, setup instructions, and history, but do not me
**Similarity:** 0.438 | **Classification:** correct

### `Q008`
**Q:** Which services are involved when a new lead submits an AutoCredit application?
**Expected:** When a new lead submits an AutoCredit application, the following services are involved in sequence: lead-intake-service captures the initial application data, eligibility-service checks basic qualific
**Actual:** Based on the available documentation, I can give a **partial answer** — the documentation only covers the Lead Intake Service in detail.

## What the documentation confirms:

When a new lead is submit
**Similarity:** 0.367 | **Classification:** correct

### `Q009`
**Q:** What is the relationship between the fraud-scoring-service and fraud-decision-service?
**Expected:** The fraud-scoring-service computes a numerical fraud risk score based on multiple signals including device fingerprinting, watchlist checks, and bureau data from ClearCheck. The fraud-decision-service
**Actual:** ## Relationship Between fraud-scoring-service and fraud-decision-service

The two services have a deliberate **separation of concerns**:

- **fraud-scoring-service** answers *"how risky is this?"* — i
**Similarity:** 0.460 | **Classification:** correct

### `Q018`
**Q:** Why does CredityFlow use a dedicated notification-template-service instead of embedding templates in the notification-service?
**Expected:** CredityFlow separates the notification-template-service to allow business and compliance teams to manage notification content independently of the delivery infrastructure. Templates can be versioned, 
**Actual:** The provided documentation does not explicitly explain the architectural rationale for separating the Notification Template Service (svc-50) from the Notification Service (svc-49). It describes *what*
**Similarity:** 0.434 | **Classification:** correct

### `Q019`
**Q:** How does the feature-flag-service integrate with LaunchDarkly to support gradual rollouts?
**Expected:** The feature-flag-service acts as an internal abstraction layer over LaunchDarkly, providing a consistent API for all CredityFlow services to query feature flags. It caches flag states in Redis to redu
**Actual:** ## Feature Flag Service & LaunchDarkly Integration

Based on the available documentation, the Feature Flag Service (`svc-52`) acts as a **caching proxy in front of LaunchDarkly**, meaning it centraliz
**Similarity:** 0.437 | **Classification:** correct

## Incorrect / Error Examples

### `Q003`
**Q:** What is the purpose of the eligibility-service in the onboarding flow?
**Expected:** The eligibility-service evaluates whether a prospective customer meets the minimum requirements for a loan product before a full proposal is created. It checks basic criteria such as age, geography, m
**Actual:** ## Purpose of the Eligibility Service

The Eligibility Service acts as the **first quality gate** in the CredityFlow onboarding pipeline. Its core purpose is to determine whether an applicant meets ba
**Similarity:** 0.364 | **Classification:** incorrect

### `Q006`
**Q:** What caching layer does CredityFlow use and which services typically rely on it?
**Expected:** CredityFlow uses Redis as its caching layer. Services that rely on Redis include the eligibility-service (for caching product rules), credit-policy-service (for caching policy configurations), feature
**Actual:** ## Caching Layer in CredityFlow (svc-18)

**CredityFlow uses Redis as its caching layer** within the Credit Bureau Integration service (svc-18).

### Cache Configuration Details
- **TTL:** 24 hours pe
**Similarity:** 0.329 | **Classification:** incorrect

### `Q007`
**Q:** Where does CredityFlow store audit logs and why was that technology chosen?
**Expected:** CredityFlow stores audit logs in Elasticsearch. This technology was chosen because audit logs require full-text search capabilities, flexible querying across multiple fields, and efficient time-series
**Actual:** ## Audit Log Storage: Elasticsearch

CredityFlow stores audit logs in **Elasticsearch**, using a **time-partitioned index pattern** (`audit-YYYY-MM`) with **1 primary shard + 1 replica** per index.

#
**Similarity:** 0.331 | **Classification:** incorrect

### `Q011`
**Q:** How does the appraisal-orchestrator-service coordinate between vehicle and property appraisals?
**Expected:** The appraisal-orchestrator-service acts as a router that determines which appraisal path to invoke based on the loan product type. For AutoCredit and AutoRefi products, it delegates to vehicle-apprais
**Actual:** ## Coordination via Unified Orchestrator Pattern

The `appraisal-orchestrator-service` handles both vehicle and property appraisals through a **single service with internal collateral-type-specific wo
**Similarity:** 0.336 | **Classification:** incorrect

### `Q016`
**Q:** Why does CredityFlow separate biometrics-capture-service from liveness-worker and face-match-service?
**Expected:** CredityFlow separates these services to follow the single-responsibility principle and enable independent scaling. The biometrics-capture-service handles real-time interaction with the customer's devi
**Actual:** 
**Similarity:** 0.000 | **Classification:** error

## Top 5 Questions by Latency

| Question ID | Latency (ms) | Classification |
|-------------|-------------|----------------|
| Q061 | 300000 | error |
| Q047 | 70963 | partial |
| Q057 | 31257 | partial |
| Q019 | 15757 | correct |
| Q011 | 15180 | incorrect |

## Top 5 Questions by Answer Divergence

| Question ID | Similarity | Classification |
|-------------|-----------|----------------|
| Q041 | 0.265 | incorrect |
| Q090 | 0.288 | incorrect |
| Q059 | 0.289 | incorrect |
| Q105 | 0.289 | incorrect |
| Q109 | 0.299 | correct |
