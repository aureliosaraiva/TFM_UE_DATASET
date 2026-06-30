# Benchmark Report — structured

**Run ID:** `structured__claude-sonnet-4-6__r3__20260623T225737-71a7c0`  
**Model:** `claude-sonnet-4-6`  
**Total questions:** 120

## Summary

| Metric | Value |
|--------|-------|
| Correct | 22 |
| Partial | 63 |
| Incorrect | 35 |
| Errors | 0 |
| Success rate | 44.6% |
| Avg latency | 8681 ms |
| Total cost | $0.8044 |

## LLM-as-Judge

| Dimension | Mean |
|-----------|------|
| correctness | 1.817 |
| completeness | 1.372 |
| groundedness | 2.528 |
| clarity | 2.961 |
| average | 2.169 |
| n (judged) | 120 |

### Judge — per category

| Category | n | correctness | completeness | groundedness | clarity | average |
|---|---|---|---|---|---|---|
| ambiguous | 15 | 1.911 | 1.378 | 2.556 | 2.911 | 2.189 |
| discovery | 18 | 1.630 | 1.148 | 2.537 | 2.926 | 2.060 |
| explanatory | 22 | 2.061 | 1.394 | 2.697 | 3.000 | 2.288 |
| factual | 24 | 1.847 | 1.389 | 2.611 | 2.958 | 2.201 |
| impact | 21 | 1.730 | 1.365 | 2.508 | 2.984 | 2.147 |
| relational | 20 | 1.700 | 1.533 | 2.233 | 2.967 | 2.108 |

## Results by Category

| Category | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| ambiguous | 3 | 12 | 0 | 0 | 15 |
| discovery | 4 | 12 | 2 | 0 | 18 |
| explanatory | 3 | 15 | 4 | 0 | 22 |
| factual | 2 | 10 | 12 | 0 | 24 |
| impact | 7 | 10 | 4 | 0 | 21 |
| relational | 3 | 4 | 13 | 0 | 20 |

## Results by Difficulty

| Difficulty | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| easy | 5 | 15 | 16 | 0 | 36 |
| hard | 10 | 25 | 9 | 0 | 44 |
| medium | 7 | 23 | 10 | 0 | 40 |

## Results by Knowledge Type

| Required Knowledge Type | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| architectural | 5 | 23 | 7 | 0 | 35 |
| factual | 3 | 12 | 11 | 0 | 26 |
| mixed | 7 | 14 | 4 | 0 | 25 |
| operational | 5 | 8 | 4 | 0 | 17 |
| relational | 2 | 6 | 9 | 0 | 17 |

## Correct Answer Examples

### `Q001`
**Q:** Which team owns the lead-intake-service?
**Expected:** The lead-intake-service is owned by team-onboarding. This team is responsible for the initial customer acquisition flow, including lead capture, eligibility checks, and proposal creation.
**Actual:** **team-onboarding** owns the `lead-intake-service`.

**Supporting details from the graph:**
- The `lead-intake-service` has an incoming relationship: `team-onboarding → OWNS`
- This is also confirmed 
**Similarity:** 0.410 | **Classification:** correct

### `Q010`
**Q:** Which services publish events that the notification-service subscribes to?
**Expected:** The notification-service subscribes to events from many services across the lending pipeline, including: proposal-service (proposal_created, proposal_approved), fraud-decision-service (fraud_cleared, 
**Actual:** ## Services That Publish Events Consumed by notification-service

Based on the graph data, the **notification-service** consumes a broad list of events. However, from the retrieved graph data, I can o
**Similarity:** 0.372 | **Classification:** correct

### `Q019`
**Q:** How does the feature-flag-service integrate with LaunchDarkly to support gradual rollouts?
**Expected:** The feature-flag-service acts as an internal abstraction layer over LaunchDarkly, providing a consistent API for all CredityFlow services to query feature flags. It caches flag states in Redis to redu
**Actual:** The **feature-flag-service does not integrate with LaunchDarkly** based on the available graph data.

## What the Graph Data Actually Shows

The feature-flag-service is a **self-contained, internally 
**Similarity:** 0.424 | **Classification:** correct

### `Q021`
**Q:** Why does the collections domain include both a negotiation-engine-service and a renegotiation-simulation-service?
**Expected:** The negotiation-engine-service and renegotiation-simulation-service serve different purposes in the collections workflow. The negotiation-engine-service handles real-time negotiation with delinquent c
**Actual:** ## Two Distinct Services for Different Collections Scenarios

Based on the service graph, the Collections domain includes both services because they address **fundamentally different borrower situatio
**Similarity:** 0.382 | **Classification:** correct

### `Q022`
**Q:** What would happen if the ocr-worker experienced a prolonged outage?
**Expected:** A prolonged ocr-worker outage would block the entire document processing pipeline. Uploaded documents would accumulate in the SQS queue as unprocessed items. Without OCR, the document-classifier-servi
**Actual:** ## Impact of a Prolonged ocr-worker Outage

### Immediate Downstream Failure
Based on the service graph, **document-classifier-service** has a direct `DEPENDS_ON` relationship with **ocr-worker**. A p
**Similarity:** 0.388 | **Classification:** correct

## Incorrect / Error Examples

### `Q002`
**Q:** What database does the document-store-service use for storing document metadata?
**Expected:** The document-store-service uses MongoDB for storing document metadata. MongoDB was chosen because documents have variable schemas depending on document type (e.g., income proofs vs. vehicle registrati
**Actual:** Based on the service graph, **document-store-service** uses **`document_meta_db`** for storing document metadata.

**Supporting details:**
- The relationship is explicitly defined as: `document-store-
**Similarity:** 0.399 | **Classification:** incorrect

### `Q003`
**Q:** What is the purpose of the eligibility-service in the onboarding flow?
**Expected:** The eligibility-service evaluates whether a prospective customer meets the minimum requirements for a loan product before a full proposal is created. It checks basic criteria such as age, geography, m
**Actual:** ## Purpose of `eligibility-service` in the Onboarding Flow

**Direct Answer:**
The `eligibility-service` runs lightweight eligibility rules against lead data to pre-qualify applicants **before** a ful
**Similarity:** 0.343 | **Classification:** incorrect

### `Q004`
**Q:** Which external partner does CredityFlow integrate with for vehicle registry operations?
**Expected:** CredityFlow integrates with DETRAN-API for vehicle registry operations. The detran-integration-service handles communication with this external partner to register liens on vehicles used as collateral
**Actual:** ## External Partner for Vehicle Registry Operations

CredityFlow integrates with **DETRAN-API** (Department of Transit) for vehicle registry operations.

### Supporting Details from the Graph

- The *
**Similarity:** 0.336 | **Classification:** incorrect

### `Q006`
**Q:** What caching layer does CredityFlow use and which services typically rely on it?
**Expected:** CredityFlow uses Redis as its caching layer. Services that rely on Redis include the eligibility-service (for caching product rules), credit-policy-service (for caching policy configurations), feature
**Actual:** ## Caching Layer in CredityFlow

**Direct Answer:**
Based on the available graph data, the only caching layer explicitly identified is **`auth_session_cache`**, used by **`auth-service`**.

---

### S
**Similarity:** 0.272 | **Classification:** incorrect

### `Q008`
**Q:** Which services are involved when a new lead submits an AutoCredit application?
**Expected:** When a new lead submits an AutoCredit application, the following services are involved in sequence: lead-intake-service captures the initial application data, eligibility-service checks basic qualific
**Actual:** ## Services Involved in a New AutoCredit Lead Submission

Based on the structured knowledge retrieved, I can only partially answer this question.

### What the graph confirms:

**`auth-service`** is c
**Similarity:** 0.332 | **Classification:** incorrect

## Top 5 Questions by Latency

| Question ID | Latency (ms) | Classification |
|-------------|-------------|----------------|
| Q117 | 21991 | partial |
| Q016 | 19821 | incorrect |
| Q027 | 17685 | correct |
| Q081 | 17328 | incorrect |
| Q046 | 16667 | incorrect |

## Top 5 Questions by Answer Divergence

| Question ID | Similarity | Classification |
|-------------|-----------|----------------|
| Q081 | 0.178 | incorrect |
| Q082 | 0.193 | incorrect |
| Q016 | 0.246 | incorrect |
| Q018 | 0.249 | incorrect |
| Q091 | 0.250 | incorrect |
