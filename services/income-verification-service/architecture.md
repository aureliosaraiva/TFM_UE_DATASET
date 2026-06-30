# Income Verification Service (svc-19)

**Domain:** Credit
**Stack:** Kotlin / Spring Boot
**Database:** PostgreSQL (`income_verification_db`)
**Deployment:** AWS EKS

## Component Overview

Income verification is one of the most consequential steps in the lending pipeline. Overstating income is the most common form of application fraud in Brazilian secured lending, and the Income Verification Service exists to catch it. This service cross-references the income declared by the applicant against two independent sources: the credit bureau report (from svc-18) and the documentary evidence (payslips, tax returns) processed by the Documents domain.

It does not simply check whether the numbers match. Real-world income verification is messy. A payslip might show gross vs. net income. A bureau report might reflect last year's data. A self-employed applicant might have irregular income across months. The service encodes these nuances into a set of verification rules that produce a confidence-weighted income estimate.

### Components

**Data Assembler:** Gathers the inputs needed for verification:
- Declared income from the proposal (fetched via Proposal Service API).
- Bureau-reported income estimate from the `CreditBureauReportReady` event.
- OCR-extracted income figures from document events (`DocumentOCRCompleted`).
The assembler waits for all inputs, with a 120-second timeout.

**Verification Engine:** Applies a series of cross-reference rules:
- Rule 1: If declared income exceeds bureau estimate by more than 40%, flag as `HIGH_DISCREPANCY`.
- Rule 2: If documentary evidence (payslip OCR) is within 10% of declared income, boost confidence.
- Rule 3: For self-employed applicants, average the last 6 months of bank statement income (if available) and compare.
- Rule 4: If multiple income sources are declared, each must have at least one supporting document.
Rules produce a composite result: `VERIFIED`, `PARTIALLY_VERIFIED`, `UNVERIFIED`, or `DISCREPANCY_DETECTED`.

**Income Estimator:** When sources disagree, this component computes a "verified income" figure -- a conservative estimate based on the most reliable source. The credit scoring engine downstream uses this figure, not the applicant's declared income, for affordability calculations.

**Result Persister and Publisher:** Stores the verification result, all input values, rule evaluations, and the final verified income in PostgreSQL. Publishes `IncomeVerificationCompleted` to SNS.

## Data Flow

1. `CreditBureauReportReady` and `DocumentOCRCompleted` events arrive asynchronously. The Data Assembler correlates them by proposal ID.
2. Once all inputs are present, the Verification Engine runs the cross-reference rules.
3. The Income Estimator computes the verified income figure.
4. Results are persisted in `income_verifications` with full audit detail.
5. `IncomeVerificationCompleted` event is published, containing the verification status and verified income.
6. Credit Scoring Engine and Credit Policy Service consume this event.

## Integration Patterns

- **Inbound (async):** SQS queues for `CreditBureauReportReady` and `DocumentOCRCompleted` events.
- **Inbound (sync):** REST call to `proposal-service` to fetch declared income.
- **Outbound (async):** `IncomeVerificationCompleted` events to SNS.
- **PostgreSQL:** Audit-grade storage of all verification inputs, rule evaluations, and outputs.

## ADR-001: Conservative Income Estimation When Sources Disagree

**Context:** The product team debated what to do when the declared income, bureau estimate, and document evidence diverge. Options included: reject the application outright, use the highest value, use the average, or use the lowest.

**Decision:** Use a conservative estimation algorithm that weights sources by reliability. Bureau data is weighted highest (0.5), document evidence second (0.35), and declared income lowest (0.15). The resulting "verified income" is used for all downstream affordability calculations. If the verified income is less than 70% of the declared income, the discrepancy is flagged for manual review.

**Consequences:**
- Lower risk of income-inflated approvals.
- Some legitimate applicants with complex income situations (freelancers, multiple jobs) may see their income underestimated, potentially affecting loan amounts. The manual review path handles these cases.
- The weighting factors are configurable and reviewed quarterly.

## ADR-002: Waiting for All Signals vs. Progressive Verification

**Context:** Income verification depends on inputs from two other domains (Credit Bureau and Documents). These arrive at unpredictable times. The team considered two approaches: wait for everything (simpler but slower) or verify progressively as each signal arrives (faster feedback but more complex).

**Decision:** Wait for all signals with a 120-second timeout. If a signal is missing after 120 seconds, proceed with available data and mark the verification as `PARTIAL`. This approach was chosen because income verification is a one-time step per proposal and the 120-second wait is invisible to the customer (it happens in the background while they complete other onboarding steps).

**Consequences:**
- Simpler implementation with a single verification pass.
- In rare timeout cases, partial verification may result in a more conservative income estimate, but the credit policy rules account for this.
- No cascading retries or complex state management needed.
