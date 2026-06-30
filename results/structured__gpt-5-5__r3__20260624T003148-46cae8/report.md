# Benchmark Report — structured

**Run ID:** `structured__gpt-5-5__r3__20260624T003148-46cae8`  
**Model:** `gpt-5.5`  
**Total questions:** 120

## Summary

| Metric | Value |
|--------|-------|
| Correct | 47 |
| Partial | 63 |
| Incorrect | 10 |
| Errors | 0 |
| Success rate | 65.4% |
| Avg latency | 5543 ms |
| Total cost | $1.0799 |

## LLM-as-Judge

| Dimension | Mean |
|-----------|------|
| correctness | 1.792 |
| completeness | 1.208 |
| groundedness | 2.639 |
| clarity | 2.903 |
| average | 2.135 |
| n (judged) | 120 |

### Judge — per category

| Category | n | correctness | completeness | groundedness | clarity | average |
|---|---|---|---|---|---|---|
| ambiguous | 15 | 1.778 | 1.089 | 2.622 | 2.800 | 2.072 |
| discovery | 18 | 1.611 | 1.111 | 2.704 | 2.907 | 2.083 |
| explanatory | 22 | 2.121 | 1.364 | 2.773 | 2.970 | 2.307 |
| factual | 24 | 1.667 | 1.208 | 2.556 | 2.861 | 2.073 |
| impact | 21 | 1.762 | 1.111 | 2.746 | 2.921 | 2.135 |
| relational | 20 | 1.783 | 1.317 | 2.433 | 2.933 | 2.117 |

## Results by Category

| Category | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| ambiguous | 11 | 4 | 0 | 0 | 15 |
| discovery | 7 | 10 | 1 | 0 | 18 |
| explanatory | 6 | 15 | 1 | 0 | 22 |
| factual | 7 | 14 | 3 | 0 | 24 |
| impact | 8 | 13 | 0 | 0 | 21 |
| relational | 8 | 7 | 5 | 0 | 20 |

## Results by Difficulty

| Difficulty | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| easy | 15 | 16 | 5 | 0 | 36 |
| hard | 18 | 23 | 3 | 0 | 44 |
| medium | 14 | 24 | 2 | 0 | 40 |

## Results by Knowledge Type

| Required Knowledge Type | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| architectural | 11 | 21 | 3 | 0 | 35 |
| factual | 9 | 14 | 3 | 0 | 26 |
| mixed | 12 | 11 | 2 | 0 | 25 |
| operational | 8 | 9 | 0 | 0 | 17 |
| relational | 7 | 8 | 2 | 0 | 17 |

## Correct Answer Examples

### `Q001`
**Q:** Which team owns the lead-intake-service?
**Expected:** The lead-intake-service is owned by team-onboarding. This team is responsible for the initial customer acquisition flow, including lead capture, eligibility checks, and proposal creation.
**Actual:** Direct answer: **team-onboarding** owns **lead-intake-service**.

Supporting details from the graph data:
- **lead-intake-service** has **Owner team: team-onboarding**
- Incoming relationship: **team-
**Similarity:** 0.473 | **Classification:** correct

### `Q002`
**Q:** What database does the document-store-service use for storing document metadata?
**Expected:** The document-store-service uses MongoDB for storing document metadata. MongoDB was chosen because documents have variable schemas depending on document type (e.g., income proofs vs. vehicle registrati
**Actual:** Direct answer: **document-store-service uses `document_meta_db` for document metadata**.

Supporting details from the graph data:
- `document-store-service → USES_DATABASE → document_meta_db`
- `docum
**Similarity:** 0.458 | **Classification:** correct

### `Q003`
**Q:** What is the purpose of the eligibility-service in the onboarding flow?
**Expected:** The eligibility-service evaluates whether a prospective customer meets the minimum requirements for a loan product before a full proposal is created. It checks basic criteria such as age, geography, m
**Actual:** The `eligibility-service` runs lightweight eligibility rules against lead data to pre-qualify applicants before full credit analysis.

Supporting details from the graph data:
- Service: `eligibility-s
**Similarity:** 0.426 | **Classification:** correct

### `Q004`
**Q:** Which external partner does CredityFlow integrate with for vehicle registry operations?
**Expected:** CredityFlow integrates with DETRAN-API for vehicle registry operations. The detran-integration-service handles communication with this external partner to register liens on vehicles used as collateral
**Actual:** CredityFlow integrates with **DETRAN-API** for vehicle registry operations.

Supporting details from the graph data:
- **detran-integration-service** has purpose: integrates with state **DETRAN** syst
**Similarity:** 0.527 | **Classification:** correct

### `Q009`
**Q:** What is the relationship between the fraud-scoring-service and fraud-decision-service?
**Expected:** The fraud-scoring-service computes a numerical fraud risk score based on multiple signals including device fingerprinting, watchlist checks, and bureau data from ClearCheck. The fraud-decision-service
**Actual:** The `fraud-decision-service` depends on the `fraud-scoring-service`.

Supporting details from the graph data:
- `fraud-scoring-service` publishes `fraud_scored`.
- `fraud-decision-service` consumes `f
**Similarity:** 0.464 | **Classification:** correct

## Incorrect / Error Examples

### `Q006`
**Q:** What caching layer does CredityFlow use and which services typically rely on it?
**Expected:** CredityFlow uses Redis as its caching layer. Services that rely on Redis include the eligibility-service (for caching product rules), credit-policy-service (for caching policy configurations), feature
**Actual:** CredityFlow uses **Redis** as a caching layer, specifically shown as **auth_session_cache** for `auth-service`.

Supporting details from the graph data:
- `auth-service` has `Database type: PostgreSQL
**Similarity:** 0.370 | **Classification:** incorrect

### `Q059`
**Q:** Trace the flow of customer PII data from initial application through disbursement. Which services handle PII and where is it stored?
**Expected:** Customer PII enters through the customer-portal-bff during application submission and is stored in the onboarding-service's database. The document-processing-service handles identity documents contain
**Actual:** Direct answer: The graph does not contain enough information to trace full customer PII flow from initial application through disbursement. It only shows disbursement-domain services, database usage, 
**Similarity:** 0.339 | **Classification:** incorrect

### `Q081`
**Q:** Trace the full sequence of services a secured vehicle loan passes through from lead capture to disbursement.
**Expected:** A vehicle loan flows through: lead-intake-service → eligibility-service → proposal-service → document-service → biometrics-service → fraud-scoring-service → credit-scoring-engine → vehicle-appraisal-s
**Actual:** The graph data is insufficient to trace the **full sequence from lead capture to disbursement**. It only contains Disbursement-domain services and a few external dependencies.

Supporting sequence ava
**Similarity:** 0.280 | **Classification:** incorrect

### `Q082`
**Q:** Which services handle PII data, and what is the flow of personally identifiable information across the platform?
**Expected:** PII flows through lead-intake-service (initial capture), customer-service (master record), proposal-service (application details), document-service (ID documents), biometrics-service (facial/fingerpri
**Actual:** Direct answer: The graph explicitly marks **auth-service** and **customer-service** as services that handle PII. The PII flow is only partially defined: **customer-service** is the canonical source of
**Similarity:** 0.225 | **Classification:** incorrect

### `Q091`
**Q:** What is the notification delivery architecture at CredityFlow?
**Expected:** Notification-service is called by many services across the platform to send notifications to customers. It depends on notification-template-service for message templates and customer-service for custo
**Actual:** The notification delivery architecture at CredityFlow is centered on `notification-service`, a Kotlin/Spring Boot service in the `Notifications` domain that sends multi-channel notifications and track
**Similarity:** 0.253 | **Classification:** incorrect

## Top 5 Questions by Latency

| Question ID | Latency (ms) | Classification |
|-------------|-------------|----------------|
| Q082 | 14532 | incorrect |
| Q087 | 14479 | partial |
| Q059 | 14238 | incorrect |
| Q095 | 12832 | correct |
| Q072 | 11468 | partial |

## Top 5 Questions by Answer Divergence

| Question ID | Similarity | Classification |
|-------------|-----------|----------------|
| Q082 | 0.225 | incorrect |
| Q091 | 0.253 | incorrect |
| Q081 | 0.280 | incorrect |
| Q115 | 0.311 | incorrect |
| Q095 | 0.318 | correct |
