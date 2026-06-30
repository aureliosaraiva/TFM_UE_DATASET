# CredityFlow Event Catalog

## Overview

CredityFlow uses an **event-driven architecture** built on Amazon SNS topics and SQS queues. Events flow through the entire lending pipeline, from lead capture through eligibility, credit analysis, contract signing, disbursement, and collections.

The platform relies on two messaging patterns:

- **SNS-SQS fan-out** -- The primary pattern. An SNS topic broadcasts each event to multiple SQS subscriber queues, allowing independent services to react in parallel without coupling.
- **Direct SQS (point-to-point)** -- Used for internal pipeline steps where exactly one consumer must process the message (e.g., OCR processing, liveness detection).

All 64 domain events listed in this catalog share a common envelope, carry a correlation ID for end-to-end tracing, and are published with at-least-once delivery semantics.

---

## Event Naming Convention

Events follow **snake_case** naming with the pattern:

```
{entity}_{past_tense_action}
```

Examples: `lead_created`, `contract_signed`, `disbursement_done`.

All events include a **correlation ID** -- typically `proposal_id` (or `lead_id` during onboarding before a proposal exists) -- that threads through every service interaction so that a single lending request can be traced from intake to disbursement and beyond.

---

## Standard Event Envelope

Every event published on CredityFlow conforms to the following envelope schema:

```json
{
  "event_id": "uuid-v4",
  "event_name": "string",
  "version": "1.0",
  "timestamp": "2026-03-29T14:22:00.000Z",
  "correlation_id": "proposal_id or lead_id",
  "producer": "service-name",
  "payload": { }
}
```

| Field            | Type   | Description                                                                 |
|------------------|--------|-----------------------------------------------------------------------------|
| `event_id`       | UUID   | Globally unique identifier for this event instance.                         |
| `event_name`     | String | Snake_case event name matching this catalog.                                |
| `version`        | String | Schema version. Consumers must tolerate unknown fields for forward compat.  |
| `timestamp`      | String | ISO-8601 UTC timestamp of when the event was produced.                      |
| `correlation_id` | String | `proposal_id` (or `lead_id` pre-proposal) for distributed tracing.         |
| `producer`       | String | Name of the service that emitted the event.                                 |
| `payload`        | Object | Domain-specific data. Structure varies per event (see tables below).        |

---

## Events by Domain

### 1. Onboarding (4 events)

| Event Name | Producer | Consumers | Trigger | Key Payload Fields |
|---|---|---|---|---|
| `lead_created` | lead-intake-service | eligibility-service, notification-service, audit-service | Customer submits initial application form | `lead_id`, `customer_name`, `cpf`, `email`, `phone`, `product_type`, `requested_amount` |
| `lead_qualified` | eligibility-service | customer-service, notification-service | Lead passes basic eligibility rules (age, region, minimum income) | `lead_id`, `eligibility_status`, `eligible_products[]`, `pre_approved_limit` |
| `proposal_started` | proposal-service | document-upload-service, notification-service, audit-service | Qualified lead confirms intent; proposal record is created | `proposal_id`, `lead_id`, `customer_id`, `product_type`, `requested_amount`, `term_months` |
| `proposal_updated` | proposal-service | notification-service | Proposal status changes at any pipeline stage | `proposal_id`, `previous_status`, `new_status`, `updated_fields[]` |

### 2. Document Management (5 events)

| Event Name | Producer | Consumers | Trigger | Key Payload Fields |
|---|---|---|---|---|
| `documents_requested` | document-upload-service | notification-service, customer-portal-bff | Proposal started; required document list is determined | `proposal_id`, `required_documents[]`, `upload_deadline` |
| `documents_uploaded` | document-upload-service | ocr-worker, audit-service | Customer uploads one or more files via portal or mobile | `proposal_id`, `document_id`, `document_type`, `file_key`, `mime_type`, `file_size_bytes` |
| `documents_classified` | document-classifier-service | document-validation-service | OCR + ML pipeline classifies the document type and extracts fields | `proposal_id`, `document_id`, `classified_type`, `confidence_score`, `extracted_fields{}` |
| `documents_validated` | document-validation-service | document-store-service, proposal-service | Document passes validation rules (readability, expiry, data consistency) | `proposal_id`, `document_id`, `validation_result`, `issues[]` |
| `documents_submitted` | document-store-service | biometrics-capture-service, notification-service | All required documents are validated and persisted | `proposal_id`, `document_bundle_id`, `document_count`, `stored_at` |

### 3. Biometrics (5 events)

| Event Name | Producer | Consumers | Trigger | Key Payload Fields |
|---|---|---|---|---|
| `biometrics_requested` | biometrics-capture-service | notification-service, customer-portal-bff | Documents submitted; biometric capture is next in pipeline | `proposal_id`, `biometrics_session_id`, `capture_type`, `expiry` |
| `biometrics_captured` | biometrics-capture-service | liveness-worker | Customer completes selfie/video capture | `proposal_id`, `biometrics_session_id`, `capture_file_key`, `device_metadata{}` |
| `liveness_verified` | liveness-worker | face-match-service | Liveness detection confirms real person (not photo/video replay) | `proposal_id`, `liveness_score`, `liveness_result` |
| `face_match_completed` | face-match-service | biometrics-capture-service | Selfie matched against document photo | `proposal_id`, `match_score`, `match_result`, `reference_document_id` |
| `biometrics_validated` | biometrics-capture-service | device-fingerprint-service, fraud-scoring-service, notification-service, audit-service | All biometric checks pass (liveness + face match) | `proposal_id`, `biometrics_session_id`, `overall_result`, `liveness_score`, `face_match_score` |

### 4. Fraud Prevention (8 events)

| Event Name | Producer | Consumers | Trigger | Key Payload Fields |
|---|---|---|---|---|
| `fraud_check_started` | fraud-scoring-service | audit-service | Biometrics validated; fraud pipeline begins | `proposal_id`, `check_id`, `checks_requested[]` |
| `device_fingerprint_checked` | device-fingerprint-service | fraud-scoring-service | Device metadata analyzed against known fraud patterns | `proposal_id`, `device_id`, `risk_level`, `fingerprint_hash`, `known_device` |
| `watchlist_checked` | watchlist-service | fraud-scoring-service | Customer screened against PEP, sanctions, and internal blacklists | `proposal_id`, `watchlist_sources[]`, `match_found`, `match_details[]` |
| `fraud_bureau_checked` | fraud-bureau-integration | fraud-scoring-service | External fraud bureau (e.g., Quod, TransUnion) returns risk data | `proposal_id`, `bureau_name`, `bureau_score`, `alerts[]` |
| `fraud_scored` | fraud-scoring-service | fraud-decision-service | All fraud signals aggregated into a composite score | `proposal_id`, `fraud_score`, `risk_band`, `signal_breakdown{}` |
| `fraud_decision_made` | fraud-decision-service | proposal-service, audit-service | Decision engine evaluates fraud score against policy thresholds | `proposal_id`, `decision`, `reason_codes[]`, `requires_manual_review` |
| `fraud_cleared` | fraud-decision-service | credit-bureau-integration, credit-scoring-engine, notification-service | Fraud decision is PASS; proposal advances to credit analysis | `proposal_id`, `fraud_score`, `cleared_at` |
| `fraud_flagged` | fraud-decision-service | case-management-service, notification-service, audit-service | Fraud decision is FAIL or REVIEW; manual investigation required | `proposal_id`, `fraud_score`, `flag_reasons[]`, `recommended_action` |

### 5. Credit Analysis (8 events)

| Event Name | Producer | Consumers | Trigger | Key Payload Fields |
|---|---|---|---|---|
| `credit_check_started` | credit-scoring-engine | audit-service | Fraud cleared; credit analysis pipeline begins | `proposal_id`, `check_id`, `bureau_requests[]` |
| `bureau_data_fetched` | credit-bureau-integration | credit-scoring-engine, income-verification-service | Credit bureau (Serasa, SPC) returns tradeline and score data | `proposal_id`, `bureau_name`, `bureau_score`, `open_accounts`, `total_debt`, `delinquencies` |
| `income_verified` | income-verification-service | credit-scoring-engine | Income cross-referenced with bureau data, bank statements, or payroll | `proposal_id`, `declared_income`, `verified_income`, `verification_method`, `confidence` |
| `credit_scored` | credit-scoring-engine | credit-policy-service | Internal scoring model produces a credit score | `proposal_id`, `internal_score`, `pd_estimate`, `score_factors[]` |
| `credit_policy_evaluated` | credit-policy-service | credit-decision-service | Score evaluated against product-specific credit policies | `proposal_id`, `policy_id`, `policy_version`, `policy_result`, `max_approved_amount`, `approved_rate` |
| `credit_decision_made` | credit-decision-service | proposal-service, audit-service | Final credit decision rendered (approved, denied, or conditional) | `proposal_id`, `decision`, `approved_amount`, `approved_rate`, `term_months`, `conditions[]` |
| `credit_approved` | credit-decision-service | appraisal-orchestrator-service, notification-service | Credit decision is APPROVED; collateral appraisal is next | `proposal_id`, `approved_amount`, `approved_rate`, `term_months` |
| `credit_denied` | credit-decision-service | notification-service, audit-service | Credit decision is DENIED | `proposal_id`, `denial_reasons[]`, `cooldown_period_days` |

### 6. Appraisal (4 events)

| Event Name | Producer | Consumers | Trigger | Key Payload Fields |
|---|---|---|---|---|
| `appraisal_requested` | appraisal-orchestrator-service | vehicle-appraisal-service, property-appraisal-service | Credit approved for a secured product; collateral must be appraised | `proposal_id`, `collateral_type`, `collateral_details{}` |
| `vehicle_appraised` | vehicle-appraisal-service | appraisal-report-service | Vehicle valued via FIPE table + condition inspection | `proposal_id`, `vehicle_id`, `fipe_value`, `appraised_value`, `condition_grade` |
| `property_appraised` | property-appraisal-service | appraisal-report-service | Property valued via comparable sales + on-site inspection | `proposal_id`, `property_id`, `appraised_value`, `ltv_ratio`, `inspection_date` |
| `appraisal_completed` | appraisal-report-service | contract-generation-service, proposal-service, notification-service | Final appraisal report consolidated; collateral value confirmed | `proposal_id`, `collateral_type`, `final_appraised_value`, `ltv_ratio`, `report_url` |

### 7. Contract (5 events)

| Event Name | Producer | Consumers | Trigger | Key Payload Fields |
|---|---|---|---|---|
| `contract_generation_requested` | contract-generation-service | audit-service | Appraisal completed; contract drafting begins | `proposal_id`, `contract_template_id`, `product_type`, `terms{}` |
| `contract_generated` | contract-generation-service | compliance-check-service | Contract PDF rendered from template with proposal terms | `proposal_id`, `contract_id`, `contract_url`, `page_count` |
| `compliance_checked` | compliance-check-service | digital-signature-service, audit-service | Contract passes regulatory compliance validation (Bacen, CDC) | `proposal_id`, `contract_id`, `compliance_result`, `checked_rules[]` |
| `signature_requested` | digital-signature-service | notification-service, customer-portal-bff | Contract ready for customer signature via ICP-Brasil or e-signature | `proposal_id`, `contract_id`, `signature_url`, `expiry` |
| `contract_signed` | digital-signature-service | contract-store-service, registry-orchestrator-service, notification-service, audit-service | Customer digitally signs the contract | `proposal_id`, `contract_id`, `signature_id`, `signer_cpf`, `signed_at`, `certificate_chain` |

### 8. Registry (5 events)

| Event Name | Producer | Consumers | Trigger | Key Payload Fields |
|---|---|---|---|---|
| `registry_requested` | registry-orchestrator-service | detran-integration-service, cartorio-integration-service, audit-service | Contract signed; collateral lien must be registered with authorities | `proposal_id`, `contract_id`, `collateral_type`, `registry_targets[]` |
| `detran_lien_registered` | detran-integration-service | registry-tracker-service | Vehicle lien registered with state DETRAN (traffic authority) | `proposal_id`, `vehicle_plate`, `renavam`, `lien_protocol`, `registered_at` |
| `cartorio_lien_registered` | cartorio-integration-service | registry-tracker-service | Property lien registered with cartorio (notary office) | `proposal_id`, `property_matricula`, `cartorio_id`, `lien_protocol`, `registered_at` |
| `registry_completed` | registry-tracker-service | disbursement-service, notification-service, audit-service | All required liens successfully registered | `proposal_id`, `contract_id`, `registrations[]`, `completed_at` |
| `registry_failed` | registry-tracker-service | case-management-service, notification-service | Lien registration rejected or timed out | `proposal_id`, `contract_id`, `failed_registry`, `failure_reason`, `retry_count` |

### 9. Disbursement (4 events)

| Event Name | Producer | Consumers | Trigger | Key Payload Fields |
|---|---|---|---|---|
| `disbursement_requested` | disbursement-service | bank-transfer-integration, audit-service | Registry completed; funds can be released | `proposal_id`, `disbursement_id`, `amount`, `destination_bank`, `destination_account`, `pix_key` |
| `bank_transfer_initiated` | bank-transfer-integration | payment-confirmation-service | TED/PIX transfer submitted to banking partner | `proposal_id`, `disbursement_id`, `transfer_id`, `transfer_type`, `initiated_at` |
| `bank_transfer_confirmed` | payment-confirmation-service | disbursement-service | Banking partner confirms funds settled in destination account | `proposal_id`, `disbursement_id`, `transfer_id`, `confirmed_at`, `settlement_proof` |
| `disbursement_done` | disbursement-service | billing-service, notification-service, audit-service, proposal-service | Disbursement lifecycle complete; loan is now active | `proposal_id`, `disbursement_id`, `amount`, `disbursed_at`, `first_installment_due` |

### 10. Collections (11 events)

| Event Name | Producer | Consumers | Trigger | Key Payload Fields |
|---|---|---|---|---|
| `installment_created` | billing-service | notification-service | Disbursement done; amortization schedule generated | `proposal_id`, `installment_id`, `installment_number`, `amount`, `due_date` |
| `installment_due` | billing-service | notification-service, customer-portal-bff | Scheduled reminder fired N days before due date | `proposal_id`, `installment_id`, `amount`, `due_date`, `days_until_due` |
| `overdue_installment` | billing-service | dunning-service, collections-contact-service, audit-service | Payment not received by due date + grace period | `proposal_id`, `installment_id`, `amount`, `due_date`, `days_overdue`, `penalty_amount` |
| `dunning_started` | dunning-service | notification-service, audit-service | Automated dunning sequence initiated for overdue account | `proposal_id`, `dunning_id`, `dunning_level`, `channel_sequence[]` |
| `collections_contact_attempted` | collections-contact-service | case-management-service, audit-service | Agent or automated system attempts contact with debtor | `proposal_id`, `contact_id`, `channel`, `outcome`, `next_action` |
| `negotiation_offered` | negotiation-engine-service | notification-service, customer-portal-bff | System generates a settlement or restructuring offer | `proposal_id`, `offer_id`, `offer_type`, `original_debt`, `offered_amount`, `discount_pct`, `expiry` |
| `negotiation_accepted` | negotiation-engine-service | billing-service, notification-service, audit-service | Customer accepts a negotiation offer | `proposal_id`, `offer_id`, `accepted_amount`, `new_due_date`, `accepted_at` |
| `legal_action_started` | legal-collections-service | case-management-service, notification-service, audit-service | Debt escalated to legal proceedings after exhausting amicable efforts | `proposal_id`, `legal_case_id`, `action_type`, `court`, `filing_date` |
| `renegotiation_requested` | renegotiation-eligibility-service | renegotiation-simulation-service, audit-service | Customer or agent requests full loan renegotiation | `proposal_id`, `request_id`, `reason`, `requested_terms{}` |
| `renegotiation_eligible` | renegotiation-eligibility-service | renegotiation-simulation-service, notification-service | Customer meets eligibility criteria for renegotiation | `proposal_id`, `request_id`, `eligible_options[]`, `max_term_extension` |
| `renegotiation_simulated` | renegotiation-simulation-service | notification-service, customer-portal-bff | Simulation engine produces renegotiation scenarios for customer review | `proposal_id`, `simulation_id`, `scenarios[]`, `best_offer{}` |

### 11. Platform (5 events)

| Event Name | Producer | Consumers | Trigger | Key Payload Fields |
|---|---|---|---|---|
| `notification_requested` | notification-service | _(internal processing)_ | Any service publishes an event that triggers a customer-facing communication | `proposal_id`, `notification_id`, `channel`, `template_id`, `recipient`, `params{}` |
| `notification_sent` | notification-service | audit-service | Email, SMS, push, or WhatsApp message successfully delivered | `proposal_id`, `notification_id`, `channel`, `delivered_at`, `provider_message_id` |
| `case_created` | case-management-service | notification-service, audit-service | Manual review case opened (fraud flag, registry failure, escalation) | `proposal_id`, `case_id`, `case_type`, `priority`, `assigned_team` |
| `case_resolved` | case-management-service | notification-service, audit-service | Case closed with resolution | `proposal_id`, `case_id`, `resolution`, `resolved_by`, `resolved_at` |
| `audit_event_logged` | audit-service | _(terminal -- no consumers)_ | Any auditable event persisted to the immutable audit log | `proposal_id`, `audit_id`, `source_event_name`, `actor`, `logged_at` |

---

## Happy Path Event Flow

The following numbered sequence represents the complete happy path for a secured lending product (e.g., vehicle-backed loan) from lead to disbursement:

1. **`lead_created`** -- Customer submits application via website or mobile app.
2. **`lead_qualified`** -- Eligibility service confirms the lead meets minimum criteria.
3. **`proposal_started`** -- Customer confirms intent; a proposal record is created.
4. **`documents_requested`** -- Required document list sent to customer.
5. **`documents_uploaded`** -- Customer uploads identity, income, and collateral documents.
6. **`documents_classified`** -- ML pipeline classifies and extracts data from each document.
7. **`documents_validated`** -- Documents pass completeness and consistency checks.
8. **`documents_submitted`** -- Full document bundle persisted and ready for next stage.
9. **`biometrics_requested`** -- Customer prompted to complete selfie capture.
10. **`biometrics_captured`** -- Selfie/video submitted for analysis.
11. **`liveness_verified`** -- Liveness detection confirms real person.
12. **`face_match_completed`** -- Selfie matches the photo on the identity document.
13. **`biometrics_validated`** -- All biometric checks pass.
14. **`fraud_check_started`** -- Fraud analysis pipeline begins.
15. **`device_fingerprint_checked`** -- Device metadata screened.
16. **`watchlist_checked`** -- Customer screened against watchlists.
17. **`fraud_bureau_checked`** -- External fraud bureau returns risk data.
18. **`fraud_scored`** -- Composite fraud score calculated.
19. **`fraud_decision_made`** -- Decision engine evaluates the score.
20. **`fraud_cleared`** -- Fraud check passes; proposal advances.
21. **`credit_check_started`** -- Credit analysis pipeline begins.
22. **`bureau_data_fetched`** -- Credit bureau returns tradeline data.
23. **`income_verified`** -- Income cross-referenced and confirmed.
24. **`credit_scored`** -- Internal credit model produces a score.
25. **`credit_policy_evaluated`** -- Score evaluated against product policies.
26. **`credit_decision_made`** -- Final credit decision rendered.
27. **`credit_approved`** -- Credit approved; collateral appraisal initiated.
28. **`appraisal_requested`** -- Collateral sent for valuation.
29. **`vehicle_appraised`** _(or `property_appraised`)_ -- Collateral valued.
30. **`appraisal_completed`** -- Final appraisal report ready.
31. **`contract_generation_requested`** -- Contract drafting begins.
32. **`contract_generated`** -- Contract PDF rendered.
33. **`compliance_checked`** -- Contract passes regulatory validation.
34. **`signature_requested`** -- Customer prompted to sign.
35. **`contract_signed`** -- Customer signs digitally.
36. **`registry_requested`** -- Lien registration submitted to authorities.
37. **`detran_lien_registered`** _(or `cartorio_lien_registered`)_ -- Lien confirmed.
38. **`registry_completed`** -- All registrations done.
39. **`disbursement_requested`** -- Fund release initiated.
40. **`bank_transfer_initiated`** -- TED/PIX submitted.
41. **`bank_transfer_confirmed`** -- Funds settled.
42. **`disbursement_done`** -- Loan is active; billing schedule created.

> Throughout the entire flow, `proposal_updated` is emitted at each major status transition, `notification_requested` and `notification_sent` fire for customer-facing communications, and `audit_event_logged` captures every auditable action.

---

## Cross-Cutting Patterns

### Audit Trail

The **audit-service** subscribes to **23 events** spanning every domain. It maintains an immutable, append-only log that satisfies regulatory requirements (Bacen, LGPD) and supports internal investigations. Events consumed:

`lead_created`, `proposal_started`, `documents_uploaded`, `biometrics_validated`, `fraud_check_started`, `fraud_decision_made`, `fraud_flagged`, `credit_check_started`, `credit_decision_made`, `credit_denied`, `contract_generation_requested`, `compliance_checked`, `contract_signed`, `registry_requested`, `registry_completed`, `disbursement_requested`, `disbursement_done`, `overdue_installment`, `dunning_started`, `collections_contact_attempted`, `negotiation_accepted`, `legal_action_started`, `renegotiation_requested`.

### Notification Hub

The **notification-service** subscribes to **25+ events** and acts as the single gateway for all customer-facing communications (email, SMS, push notification, WhatsApp). It uses template-based rendering so business teams can update copy without code changes.

### Correlation and Tracing

All events carry `proposal_id` (or `lead_id` before proposal creation) as the `correlation_id`. Combined with `event_id` and `timestamp`, this enables:

- **End-to-end latency tracking** across the full pipeline.
- **Funnel analytics** showing drop-off at each stage.
- **Incident investigation** by querying all events for a given proposal.

### Dead Letter Queues (DLQ)

Every SQS queue has a DLQ configured with a **maxReceiveCount of 3**. Messages that fail processing three times are moved to the DLQ. An automated alarm triggers when DLQ depth exceeds zero, and the operations team triages failures via the internal dashboard.

### Idempotency

All consumers are designed to be **idempotent**. They use `event_id` as a deduplication key, ensuring that redelivered messages (inherent in at-least-once delivery) do not produce duplicate side effects.

---

## Event Volume Estimates

Estimated daily volumes for a mature CredityFlow deployment:

| Event | Est. Daily Volume | Notes |
|---|---|---|
| `lead_created` | ~5,000 | Inbound leads from all channels |
| `lead_qualified` | ~3,500 | ~70% pass basic eligibility |
| `proposal_started` | ~2,000 | ~57% of qualified leads proceed |
| `documents_uploaded` | ~6,000 | ~3 documents per proposal on average |
| `biometrics_validated` | ~1,800 | Small drop-off due to capture failures |
| `fraud_cleared` | ~1,600 | ~89% pass fraud checks |
| `fraud_flagged` | ~200 | ~11% flagged for review |
| `credit_approved` | ~800 | ~50% approval rate post-fraud |
| `credit_denied` | ~800 | Denial notices |
| `appraisal_completed` | ~700 | Some proposals are unsecured (no appraisal) |
| `contract_signed` | ~500 | Conversion drop-off at signature stage |
| `registry_completed` | ~450 | Small failure/retry rate |
| `disbursement_done` | ~200 | Final funded loans per day |
| `installment_due` | ~15,000 | Reminders across entire active portfolio |
| `overdue_installment` | ~2,500 | Based on ~5% delinquency rate |
| `notification_sent` | ~50,000 | Aggregate across all channels and triggers |
| `audit_event_logged` | ~80,000 | High volume; every auditable action captured |

---

## Summary

The CredityFlow event catalog defines **64 domain events** across **11 domains**, processed by **40+ microservices**. The architecture ensures loose coupling between services, full auditability of the lending lifecycle, and the ability to scale each stage of the pipeline independently. Event schemas are versioned and consumers are built to tolerate additive changes, enabling the platform to evolve without coordinated deployments.
