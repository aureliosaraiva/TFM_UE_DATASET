# Benchmark Report — structured

**Run ID:** `structured__grok-build-0-1__r3__20260624T014025-0815b2`  
**Model:** `grok-build-0.1`  
**Total questions:** 120

## Summary

| Metric | Value |
|--------|-------|
| Correct | 49 |
| Partial | 61 |
| Incorrect | 10 |
| Errors | 0 |
| Success rate | 66.2% |
| Avg latency | 8821 ms |
| Total cost | $0.4700 |

## LLM-as-Judge

| Dimension | Mean |
|-----------|------|
| correctness | 1.722 |
| completeness | 1.164 |
| groundedness | 2.608 |
| clarity | 2.892 |
| average | 2.096 |
| n (judged) | 120 |

### Judge — per category

| Category | n | correctness | completeness | groundedness | clarity | average |
|---|---|---|---|---|---|---|
| ambiguous | 15 | 1.622 | 0.822 | 2.644 | 2.822 | 1.978 |
| discovery | 18 | 1.537 | 1.056 | 2.611 | 2.870 | 2.018 |
| explanatory | 22 | 1.970 | 1.303 | 2.742 | 2.924 | 2.235 |
| factual | 24 | 1.764 | 1.347 | 2.625 | 2.972 | 2.177 |
| impact | 21 | 1.730 | 1.095 | 2.778 | 2.968 | 2.143 |
| relational | 20 | 1.633 | 1.217 | 2.233 | 2.750 | 1.958 |

## Results by Category

| Category | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| ambiguous | 14 | 1 | 0 | 0 | 15 |
| discovery | 7 | 11 | 0 | 0 | 18 |
| explanatory | 7 | 14 | 1 | 0 | 22 |
| factual | 3 | 14 | 7 | 0 | 24 |
| impact | 9 | 12 | 0 | 0 | 21 |
| relational | 9 | 9 | 2 | 0 | 20 |

## Results by Difficulty

| Difficulty | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| easy | 12 | 16 | 8 | 0 | 36 |
| hard | 20 | 23 | 1 | 0 | 44 |
| medium | 17 | 22 | 1 | 0 | 40 |

## Results by Knowledge Type

| Required Knowledge Type | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| architectural | 13 | 20 | 2 | 0 | 35 |
| factual | 5 | 14 | 7 | 0 | 26 |
| mixed | 13 | 12 | 0 | 0 | 25 |
| operational | 9 | 8 | 0 | 0 | 17 |
| relational | 9 | 7 | 1 | 0 | 17 |

## Correct Answer Examples

### `Q001`
**Q:** Which team owns the lead-intake-service?
**Expected:** The lead-intake-service is owned by team-onboarding. This team is responsible for the initial customer acquisition flow, including lead capture, eligibility checks, and proposal creation.
**Actual:** **Direct answer:**  
team-onboarding owns the lead-intake-service.

**Supporting details from the graph data:**  
- In the [Service] lead-intake-service definition, the field "Owner team: team-onboard
**Similarity:** 0.463 | **Classification:** correct

### `Q003`
**Q:** What is the purpose of the eligibility-service in the onboarding flow?
**Expected:** The eligibility-service evaluates whether a prospective customer meets the minimum requirements for a loan product before a full proposal is created. It checks basic criteria such as age, geography, m
**Actual:** **Direct answer:**  
The eligibility-service runs lightweight eligibility rules against lead data (age, geography, minimum income) to pre-qualify applicants before a full credit analysis, reducing unn
**Similarity:** 0.432 | **Classification:** correct

### `Q009`
**Q:** What is the relationship between the fraud-scoring-service and fraud-decision-service?
**Expected:** The fraud-scoring-service computes a numerical fraud risk score based on multiple signals including device fingerprinting, watchlist checks, and bureau data from ClearCheck. The fraud-decision-service
**Actual:** **Direct answer:**  
The `fraud-decision-service` depends on the `fraud-scoring-service`.

**Supporting details from the graph data:**  
- `fraud-decision-service` lists `fraud-scoring-service` under 
**Similarity:** 0.429 | **Classification:** correct

### `Q010`
**Q:** Which services publish events that the notification-service subscribes to?
**Expected:** The notification-service subscribes to events from many services across the lending pipeline, including: proposal-service (proposal_created, proposal_approved), fraud-decision-service (fraud_cleared, 
**Actual:** **notification-template-service and negotiation-engine-service**

**Supporting details from the graph data:**
- notification-service consumes `notification_requested`, which is published by notificati
**Similarity:** 0.424 | **Classification:** correct

### `Q011`
**Q:** How does the appraisal-orchestrator-service coordinate between vehicle and property appraisals?
**Expected:** The appraisal-orchestrator-service acts as a router that determines which appraisal path to invoke based on the loan product type. For AutoCredit and AutoRefi products, it delegates to vehicle-apprais
**Actual:** **Direct answer:**  
The appraisal-orchestrator-service coordinates between vehicle and property appraisals by routing to the appropriate appraisal service (property-appraisal-service or vehicle-appra
**Similarity:** 0.434 | **Classification:** correct

## Incorrect / Error Examples

### `Q002`
**Q:** What database does the document-store-service use for storing document metadata?
**Expected:** The document-store-service uses MongoDB for storing document metadata. MongoDB was chosen because documents have variable schemas depending on document type (e.g., income proofs vs. vehicle registrati
**Actual:** **Direct answer:**  
document_meta_db

**Supporting details from the graph data:**  
The document-store-service lists its primary database as `document_store_db` (type: MongoDB+S3) in the service defi
**Similarity:** 0.386 | **Classification:** incorrect

### `Q004`
**Q:** Which external partner does CredityFlow integrate with for vehicle registry operations?
**Expected:** CredityFlow integrates with DETRAN-API for vehicle registry operations. The detran-integration-service handles communication with this external partner to register liens on vehicles used as collateral
**Actual:** **Direct answer:**  
CredityFlow integrates with **DETRAN-API** for vehicle registry operations.

**Supporting details from the graph data:**  
- The `detran-integration-service` (Domain: Registry, Su
**Similarity:** 0.387 | **Classification:** incorrect

### `Q006`
**Q:** What caching layer does CredityFlow use and which services typically rely on it?
**Expected:** CredityFlow uses Redis as its caching layer. Services that rely on Redis include the eligibility-service (for caching product rules), credit-policy-service (for caching policy configurations), feature
**Actual:** **Direct answer:**  
The provided service graph does not describe a platform-wide caching layer for CredityFlow. It only shows that auth-service uses `auth_session_cache` (associated with Redis).

**S
**Similarity:** 0.389 | **Classification:** incorrect

### `Q052`
**Q:** How many total events flow through the CredityFlow system, and which messaging technologies facilitate this?
**Expected:** There are 64 events flowing through the CredityFlow system. These events are facilitated by AWS SQS (Simple Queue Service) for point-to-point messaging and AWS SNS (Simple Notification Service) for pu
**Actual:** **Direct answer:** The provided service graph contains no information on the total number of events flowing through the CredityFlow system or any messaging technologies.

**Supporting details from the
**Similarity:** 0.388 | **Classification:** incorrect

### `Q058`
**Q:** Which BFF (Backend-for-Frontend) services exist in CredityFlow, and what technology stack do they use?
**Expected:** CredityFlow has two BFF services: customer-portal-bff serving the customer-facing portal and backoffice-bff serving the internal backoffice application. Both BFFs use Node.js with Express framework, w
**Actual:** **Direct answer:**  
The BFF (Backend-for-Frontend) services in CredityFlow are `customer-portal-bff` and `backoffice-bff`. Both use the same technology stack: Node.js (language), Express (framework),
**Similarity:** 0.361 | **Classification:** incorrect

## Top 5 Questions by Latency

| Question ID | Latency (ms) | Classification |
|-------------|-------------|----------------|
| Q075 | 36948 | partial |
| Q069 | 28935 | partial |
| Q083 | 24495 | partial |
| Q074 | 24121 | correct |
| Q010 | 23140 | correct |

## Top 5 Questions by Answer Divergence

| Question ID | Similarity | Classification |
|-------------|-----------|----------------|
| Q091 | 0.234 | incorrect |
| Q084 | 0.284 | incorrect |
| Q081 | 0.291 | incorrect |
| Q115 | 0.292 | incorrect |
| Q075 | 0.322 | partial |
