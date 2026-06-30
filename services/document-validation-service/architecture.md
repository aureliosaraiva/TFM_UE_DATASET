# Document Validation Service (svc-08)

**Domain:** Documents
**Stack:** Kotlin / Spring Boot
**Database:** PostgreSQL (`document_validation_db`)
**Deployment:** AWS EKS

## Component Overview

Once a document has been uploaded, OCR'd, and classified, it arrives at the Document Validation Service for rule-based verification. This service answers a critical question: does the submitted document meet the requirements for this proposal type?

The validation engine is built around a pluggable rule framework. Each document type has an associated set of validation rules, and the rules themselves are stored in the database so they can be modified by the product team without code deployments.

### Core Components

**Rule Engine:** The heart of the service. Rules are defined as a combination of field checks (e.g., "CPF field must be present," "expiration date must be in the future," "income value must be a positive number"), cross-field checks (e.g., "name on document must match customer profile name"), and format checks (e.g., "CPF must pass mod-11 check digit validation"). Rules are evaluated in priority order, and evaluation short-circuits on the first critical failure.

**Rule Repository:** PostgreSQL tables that store rule definitions, organized by document type and proposal product. The backoffice team can enable, disable, or adjust rules through an admin API. Rule changes are versioned -- every modification creates a new rule version, and validations reference the rule version that was active at the time of evaluation.

**Validation Orchestrator:** Receives a validation request (either via REST or SQS), fetches the applicable rules, retrieves OCR results and classification data from upstream services, and runs the rule engine. Results are persisted and published as events.

**Cross-Reference Client:** For rules that require external data (e.g., matching the document's CPF against the customer profile), the service calls the Customer Service API synchronously during validation.

## Data Flow

1. A `DocumentClassified` event (or a `DocumentOCRCompleted` event, depending on configuration) triggers validation.
2. The service retrieves the document's OCR output and classification from the Document Store or directly from the event payload.
3. The applicable rule set is loaded from `document_validation_db` based on document type and proposal product.
4. Rules are executed sequentially. Each rule produces a `PASS`, `WARN`, or `FAIL` result with a human-readable message.
5. The aggregate result is persisted in a `validation_results` table and published as either `DocumentValidated` or `DocumentValidationFailed` to SNS.
6. The Proposal Service consumes these events to advance (or block) the proposal state machine.

## Integration Patterns

- **Inbound (async):** SQS subscriptions to `DocumentClassified` and `DocumentOCRCompleted` events.
- **Inbound (sync):** REST endpoint for on-demand re-validation triggered by backoffice.
- **Outbound (async):** `DocumentValidated` / `DocumentValidationFailed` events to SNS.
- **Outbound (sync):** Calls `customer-service` API for cross-reference checks.

## ADR-001: Database-Driven Rules vs. Code-Defined Rules

**Context:** The initial implementation hardcoded validation rules in Kotlin. Every rule change required a code change, a PR review, and a deployment. The product team requested faster turnaround for rule adjustments, especially during the early period when document requirements were evolving rapidly.

**Decision:** Move rule definitions to PostgreSQL. Rules are expressed as structured records with a field name, operator (equals, greater than, regex match, etc.), expected value, and severity (critical, warning, informational). The Kotlin rule engine interprets these records at runtime.

**Consequences:**
- Rule changes take effect immediately after database update, no deployment needed.
- Complex rules that require custom logic (e.g., mod-11 CPF validation) are still implemented in code as "built-in" rule types; the database record references the built-in rule by name.
- Risk of invalid rules being inserted is mitigated by a validation layer on the admin API and a dry-run mode.

## ADR-002: Versioned Rule Sets for Audit Trail

**Context:** Regulatory audits require the ability to reproduce exactly which rules were applied to a given document at a specific point in time. If rules are mutable, there is no way to answer this question after the fact.

**Decision:** Implement rule versioning. Every rule modification creates a new version. Validation results store the rule version ID that was active during evaluation. Rules are never deleted, only deactivated.

**Consequences:**
- Full reproducibility: any validation result can be re-evaluated using the exact rule set that was in effect.
- Increased storage, but rule data is small (hundreds of rows, not millions).
- The admin API exposes a "diff" endpoint that shows what changed between rule versions, aiding compliance reviews.
