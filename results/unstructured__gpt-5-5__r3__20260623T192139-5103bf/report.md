# Benchmark Report — unstructured

**Run ID:** `unstructured__gpt-5-5__r3__20260623T192139-5103bf`  
**Model:** `gpt-5.5`  
**Total questions:** 120

## Summary

| Metric | Value |
|--------|-------|
| Correct | 53 |
| Partial | 67 |
| Incorrect | 0 |
| Errors | 0 |
| Success rate | 72.1% |
| Avg latency | 5171 ms |
| Total cost | $1.6881 |

## LLM-as-Judge

| Dimension | Mean |
|-----------|------|
| correctness | 2.050 |
| completeness | 1.472 |
| groundedness | 2.933 |
| clarity | 2.956 |
| average | 2.353 |
| n (judged) | 120 |

### Judge — per category

| Category | n | correctness | completeness | groundedness | clarity | average |
|---|---|---|---|---|---|---|
| ambiguous | 15 | 2.000 | 1.267 | 2.911 | 2.978 | 2.289 |
| discovery | 18 | 1.907 | 1.278 | 2.926 | 2.889 | 2.250 |
| explanatory | 22 | 2.364 | 1.864 | 3.000 | 3.000 | 2.557 |
| factual | 24 | 2.236 | 1.806 | 2.889 | 2.958 | 2.472 |
| impact | 21 | 1.730 | 1.016 | 2.905 | 2.905 | 2.139 |
| relational | 20 | 1.983 | 1.450 | 2.967 | 3.000 | 2.350 |

## Results by Category

| Category | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| ambiguous | 13 | 2 | 0 | 0 | 15 |
| discovery | 5 | 13 | 0 | 0 | 18 |
| explanatory | 12 | 10 | 0 | 0 | 22 |
| factual | 7 | 17 | 0 | 0 | 24 |
| impact | 13 | 8 | 0 | 0 | 21 |
| relational | 3 | 17 | 0 | 0 | 20 |

## Results by Difficulty

| Difficulty | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| easy | 14 | 22 | 0 | 0 | 36 |
| hard | 19 | 25 | 0 | 0 | 44 |
| medium | 20 | 20 | 0 | 0 | 40 |

## Results by Knowledge Type

| Required Knowledge Type | Correct | Partial | Incorrect | Errors | Total |
|---|---|---|---|---|---|
| architectural | 15 | 20 | 0 | 0 | 35 |
| factual | 9 | 17 | 0 | 0 | 26 |
| mixed | 16 | 9 | 0 | 0 | 25 |
| operational | 9 | 8 | 0 | 0 | 17 |
| relational | 4 | 13 | 0 | 0 | 17 |

## Correct Answer Examples

### `Q001`
**Q:** Which team owns the lead-intake-service?
**Expected:** The lead-intake-service is owned by team-onboarding. This team is responsible for the initial customer acquisition flow, including lead capture, eligibility checks, and proposal creation.
**Actual:** The documentation does not state which team owns the `lead-intake-service`.

Supporting details: the provided docs describe the service purpose, tech stack, local setup, testing, and architecture, but
**Similarity:** 0.461 | **Classification:** correct

### `Q003`
**Q:** What is the purpose of the eligibility-service in the onboarding flow?
**Expected:** The eligibility-service evaluates whether a prospective customer meets the minimum requirements for a loan product before a full proposal is created. It checks basic criteria such as age, geography, m
**Actual:** The `eligibility-service` is the first quality gate in the CredityFlow onboarding pipeline. Its purpose is to check whether a new lead meets basic eligibility requirements before downstream services s
**Similarity:** 0.511 | **Classification:** correct

### `Q008`
**Q:** Which services are involved when a new lead submits an AutoCredit application?
**Expected:** When a new lead submits an AutoCredit application, the following services are involved in sequence: lead-intake-service captures the initial application data, eligibility-service checks basic qualific
**Actual:** The documentation does not mention “AutoCredit” specifically, so I can’t identify an AutoCredit-specific flow.

For a new lead submission in general, the documented services involved are:

- **Lead In
**Similarity:** 0.423 | **Classification:** correct

### `Q009`
**Q:** What is the relationship between the fraud-scoring-service and fraud-decision-service?
**Expected:** The fraud-scoring-service computes a numerical fraud risk score based on multiple signals including device fingerprinting, watchlist checks, and bureau data from ClearCheck. The fraud-decision-service
**Actual:** The **fraud-scoring-service** calculates a fraud risk score, while the **fraud-decision-service** turns that score into an action such as approve, deny, or manual review.

Supporting details:
- The fr
**Similarity:** 0.560 | **Classification:** correct

### `Q015`
**Q:** Why did CredityFlow choose event-driven choreography instead of centralized orchestration for the lending pipeline?
**Expected:** CredityFlow chose event-driven choreography for the main lending pipeline to achieve loose coupling between domains, allowing each team to develop and deploy independently. Choreography avoids a singl
**Actual:** CredityFlow’s documentation does **not explicitly state why** event-driven choreography was chosen over centralized orchestration.

What it does say is that CredityFlow uses an event-driven architectu
**Similarity:** 0.525 | **Classification:** correct

## Top 5 Questions by Latency

| Question ID | Latency (ms) | Classification |
|-------------|-------------|----------------|
| Q117 | 21170 | correct |
| Q001 | 16025 | correct |
| Q006 | 13638 | partial |
| Q059 | 11152 | partial |
| Q109 | 8959 | correct |

## Top 5 Questions by Answer Divergence

| Question ID | Similarity | Classification |
|-------------|-----------|----------------|
| Q059 | 0.396 | partial |
| Q082 | 0.401 | partial |
| Q090 | 0.402 | partial |
| Q100 | 0.406 | partial |
| Q102 | 0.406 | correct |
