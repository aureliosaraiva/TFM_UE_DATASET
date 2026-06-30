# Benchmark Report — unstructured

**Run ID:** `unstructured__grok-build-0-1__r3__20260623T205112-438013`  
**Model:** `grok-build-0.1`  
**Total questions:** 120

## Summary

| Metric | Value |
|--------|-------|
| Correct | 34 |
| Partial | 77 |
| Incorrect | 8 |
| Errors | 1 |
| Success rate | 60.4% |
| Avg latency | 8664 ms |
| Total cost | $0.7537 |

## LLM-as-Judge

| Dimension | Mean |
|-----------|------|
| correctness | 2.061 |
| completeness | 1.503 |
| groundedness | 2.944 |
| clarity | 2.964 |
| average | 2.368 |
| n (judged) | 120 |

### Judge — per category

| Category | n | correctness | completeness | groundedness | clarity | average |
|---|---|---|---|---|---|---|
| ambiguous | 15 | 1.822 | 1.133 | 2.733 | 2.800 | 2.122 |
| discovery | 18 | 2.074 | 1.407 | 2.982 | 3.000 | 2.366 |
| explanatory | 22 | 2.364 | 1.879 | 2.939 | 2.939 | 2.530 |
| factual | 24 | 2.181 | 1.861 | 2.972 | 3.000 | 2.503 |
| impact | 21 | 1.889 | 1.190 | 2.984 | 3.000 | 2.266 |
| relational | 20 | 1.933 | 1.350 | 3.000 | 3.000 | 2.321 |

## Results by Category

| Category | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| ambiguous | 10 | 4 | 0 | 1 | 15 |
| discovery | 1 | 17 | 0 | 0 | 18 |
| explanatory | 8 | 14 | 0 | 0 | 22 |
| factual | 6 | 12 | 6 | 0 | 24 |
| impact | 6 | 15 | 0 | 0 | 21 |
| relational | 3 | 15 | 2 | 0 | 20 |

## Results by Difficulty

| Difficulty | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| easy | 11 | 19 | 5 | 1 | 36 |
| hard | 11 | 32 | 1 | 0 | 44 |
| medium | 12 | 26 | 2 | 0 | 40 |

## Results by Knowledge Type

| Required Knowledge Type | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| architectural | 9 | 24 | 1 | 1 | 35 |
| factual | 7 | 14 | 5 | 0 | 26 |
| mixed | 9 | 16 | 0 | 0 | 25 |
| operational | 6 | 11 | 0 | 0 | 17 |
| relational | 3 | 12 | 2 | 0 | 17 |

## Correct Answer Examples

### `Q001`
**Q:** Which team owns the lead-intake-service?
**Expected:** The lead-intake-service is owned by team-onboarding. This team is responsible for the initial customer acquisition flow, including lead capture, eligibility checks, and proposal creation.
**Actual:** **The provided documentation does not specify which team owns the lead-intake-service.**

The context describes the service's purpose, tech stack (Kotlin / Spring Boot, PostgreSQL `lead_intake_db`), l
**Similarity:** 0.404 | **Classification:** correct

### `Q003`
**Q:** What is the purpose of the eligibility-service in the onboarding flow?
**Expected:** The eligibility-service evaluates whether a prospective customer meets the minimum requirements for a loan product before a full proposal is created. It checks basic criteria such as age, geography, m
**Actual:** **Direct answer:**  
The eligibility-service acts as the first quality gate in the CredityFlow onboarding pipeline. It evaluates whether a new lead meets basic eligibility requirements before the syst
**Similarity:** 0.435 | **Classification:** correct

### `Q008`
**Q:** Which services are involved when a new lead submits an AutoCredit application?
**Expected:** When a new lead submits an AutoCredit application, the following services are involved in sequence: lead-intake-service captures the initial application data, eligibility-service checks basic qualific
**Actual:** **Direct answer:** The provided documentation does not contain information about "AutoCredit" applications or the full set of services involved in submitting one.

**Supporting details:**  
The docume
**Similarity:** 0.482 | **Classification:** correct

### `Q009`
**Q:** What is the relationship between the fraud-scoring-service and fraud-decision-service?
**Expected:** The fraud-scoring-service computes a numerical fraud risk score based on multiple signals including device fingerprinting, watchlist checks, and bureau data from ClearCheck. The fraud-decision-service
**Actual:** **Direct answer:**  
The fraud-scoring-service produces a fraud risk score (0–1000) that is fed directly into the fraud-decision-service. The scoring service performs the ML-based risk assessment, whi
**Similarity:** 0.434 | **Classification:** correct

### `Q015`
**Q:** Why did CredityFlow choose event-driven choreography instead of centralized orchestration for the lending pipeline?
**Expected:** CredityFlow chose event-driven choreography for the main lending pipeline to achieve loose coupling between domains, allowing each team to develop and deploy independently. Choreography avoids a singl
**Actual:** **The provided documentation does not explain why CredityFlow chose event-driven choreography over centralized orchestration.**

It only describes the architecture that was implemented, without any co
**Similarity:** 0.503 | **Classification:** correct

## Incorrect / Error Examples

### `Q005`
**Q:** What API gateway does CredityFlow use to manage external-facing traffic?
**Expected:** CredityFlow uses Kong API Gateway to manage external-facing traffic. Kong handles routing, rate limiting, authentication, and request transformation at the edge. Internally, Istio service mesh manages
**Actual:** **The provided documentation does not mention any API gateway.**

The Customer Portal BFF documentation describes direct HTTPS traffic from the Customer Browser (React SPA) to the BFF itself, which th
**Similarity:** 0.294 | **Classification:** incorrect

### `Q006`
**Q:** What caching layer does CredityFlow use and which services typically rely on it?
**Expected:** CredityFlow uses Redis as its caching layer. Services that rely on Redis include the eligibility-service (for caching product rules), credit-policy-service (for caching policy configurations), feature
**Actual:** **Direct answer:**  
CredityFlow uses **Redis** as its caching layer (with a 24-hour TTL). The provided documentation only describes its use in the Credit Bureau Integration service (svc-18).

**Suppo
**Similarity:** 0.364 | **Classification:** incorrect

### `Q007`
**Q:** Where does CredityFlow store audit logs and why was that technology chosen?
**Expected:** CredityFlow stores audit logs in Elasticsearch. This technology was chosen because audit logs require full-text search capabilities, flexible querying across multiple fields, and efficient time-series
**Actual:** **Direct answer:**  
CredityFlow stores audit logs in **Elasticsearch** (indices named `audit-YYYY-MM`, configured with 1 primary + 1 replica).

**Supporting details:**  
- The Audit Service is explic
**Similarity:** 0.399 | **Classification:** incorrect

### `Q041`
**Q:** Which programming language and framework does the feature-flag-service use, and which domain does it belong to?
**Expected:** The feature-flag-service is implemented in Go using the Gin framework. It belongs to the Platform domain, which provides cross-cutting infrastructure capabilities to all other services in the CredityF
**Actual:** **The provided documentation does not explicitly state the programming language, framework, or domain for the feature-flag-service.**

The architecture description covers only high-level components (F
**Similarity:** 0.281 | **Classification:** incorrect

### `Q043`
**Q:** List all external partners that CredityFlow integrates with for vehicle-related operations.
**Expected:** For vehicle-related operations, CredityFlow integrates with DETRAN-API for vehicle registration and lien checks, FIPE-API for vehicle valuation based on the FIPE table, InspecaoExpress for vehicle ins
**Actual:** **Direct answer:**  
The documentation does not list any specific named external partners. It only describes general categories of integrations and mentions one government agency (DETRAN) relevant to 
**Similarity:** 0.383 | **Classification:** incorrect

## Top 5 Questions by Latency

| Question ID | Latency (ms) | Classification |
|-------------|-------------|----------------|
| Q080 | 300000 | error |
| Q070 | 25214 | partial |
| Q049 | 15351 | partial |
| Q012 | 15207 | partial |
| Q104 | 13809 | partial |

## Top 5 Questions by Answer Divergence

| Question ID | Similarity | Classification |
|-------------|-----------|----------------|
| Q041 | 0.281 | incorrect |
| Q005 | 0.294 | incorrect |
| Q082 | 0.316 | partial |
| Q091 | 0.321 | partial |
| Q115 | 0.341 | incorrect |
