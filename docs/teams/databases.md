# CredityFlow Database Inventory

This document catalogs all databases and data stores used across the CredityFlow platform. Each entry includes the database type, owning services, data classification, and purpose.

## Data Classification

Databases are tagged with two classification flags:

- **Contains PII** -- Stores personally identifiable information (names, CPF, addresses, biometric data, etc.). Subject to LGPD and data protection policies.
- **Contains Financial** -- Stores financial data (loan amounts, scores, payment records, account details). Subject to Central Bank regulations and financial audit requirements.

## Storage Technologies

The platform uses the following storage technologies:

| Technology | Usage |
|---|---|
| **PostgreSQL** | Primary relational store for most services |
| **MongoDB** | Document metadata and unstructured data |
| **Redis** | Caching, session management, and ephemeral processing data |
| **S3** | Object storage for files (documents, reports, signed contracts) |
| **Elasticsearch** | Audit logs and full-text search |

---

## Full Database Inventory

| Database Name | Type | Used By | PII | Financial | Purpose |
|---|---|---|---|---|---|
| `lead_intake_db` | PostgreSQL | lead-intake-service | Yes | No | Stores leads and sources |
| `eligibility_db` | PostgreSQL | eligibility-service | No | No | Eligibility rules and results |
| `customer_db` | PostgreSQL | customer-service | Yes | No | Customer profiles and contacts |
| `proposal_db` | PostgreSQL | proposal-service | Yes | Yes | Loan proposals and status |
| `document_meta_db` | MongoDB | document-upload-service, document-store-service | Yes | No | Document metadata and references |
| `document_files` | S3 | document-upload-service, document-store-service | Yes | No | Raw document files |
| `document_classification_db` | PostgreSQL | document-classifier-service | No | No | Classification models and results |
| `document_validation_db` | PostgreSQL | document-validation-service | No | No | Validation rules and results |
| `biometrics_db` | PostgreSQL | biometrics-capture-service, face-match-service | Yes | No | Biometric records and match results |
| `biometrics_cache` | Redis | liveness-worker | Yes | No | Temporary biometric processing data |
| `device_fingerprint_cache` | Redis | device-fingerprint-service | No | No | Device fingerprint hashes |
| `watchlist_db` | PostgreSQL | watchlist-service | Yes | No | Sanctions and PEP lists |
| `fraud_scores_db` | PostgreSQL | fraud-scoring-service | No | No | Fraud scores and signals |
| `fraud_decisions_db` | PostgreSQL | fraud-decision-service | No | No | Fraud decisions and overrides |
| `credit_bureau_cache` | Redis | credit-bureau-integration | Yes | Yes | Cached bureau responses |
| `income_db` | PostgreSQL | income-verification-service | Yes | Yes | Income data and verification results |
| `credit_scores_db` | PostgreSQL | credit-scoring-engine | No | Yes | Credit scores and model outputs |
| `credit_policy_db` | PostgreSQL | credit-policy-service | No | No | Policy rules and configurations |
| `credit_decisions_db` | PostgreSQL | credit-decision-service | No | Yes | Credit decisions and rationale |
| `appraisal_db` | PostgreSQL | appraisal-orchestrator-service, vehicle-appraisal-service, property-appraisal-service | No | Yes | Appraisal requests and valuations |
| `appraisal_reports` | S3 | appraisal-report-service | No | Yes | Appraisal report PDFs |
| `appraisal_report_db` | PostgreSQL | appraisal-report-service | No | Yes | Report metadata |
| `contract_db` | PostgreSQL | contract-generation-service, compliance-check-service | Yes | Yes | Contract data and terms |
| `contract_files` | MongoDB+S3 | contract-store-service | Yes | Yes | Signed contract documents |
| `signature_db` | PostgreSQL | digital-signature-service | Yes | No | Signature records and certificates |
| `registry_db` | PostgreSQL | registry-orchestrator-service, detran-integration-service, cartorio-integration-service, registry-tracker-service | No | No | Registration requests and status |
| `disbursement_db` | PostgreSQL | disbursement-service, bank-transfer-integration, payment-confirmation-service | No | Yes | Disbursement orders and confirmations |
| `billing_db` | PostgreSQL | billing-service | No | Yes | Installments and payment records |
| `dunning_db` | PostgreSQL | dunning-service | No | Yes | Dunning campaigns and contacts |
| `collections_db` | PostgreSQL | collections-contact-service, negotiation-engine-service | No | Yes | Contact history and negotiations |
| `legal_collections_db` | PostgreSQL | legal-collections-service | Yes | Yes | Legal proceedings records |
| `renegotiation_db` | PostgreSQL | renegotiation-eligibility-service, renegotiation-simulation-service | No | Yes | Renegotiation eligibility and simulations |
| `portal_session_cache` | Redis | customer-portal-bff | Yes | No | Session data for portal |
| `backoffice_session_cache` | Redis | backoffice-bff | No | No | Session data for backoffice |
| `case_management_db` | PostgreSQL | case-management-service | Yes | No | Cases, assignments, and resolution |
| `audit_log_store` | Elasticsearch | audit-service | Yes | Yes | Audit trail and compliance logs |
| `notification_db` | PostgreSQL | notification-service | Yes | No | Notification records and delivery status |
| `notification_templates_db` | PostgreSQL | notification-template-service | No | No | Message templates |
| `auth_db` | PostgreSQL | auth-service | Yes | No | Users, roles, and permissions |
| `auth_session_cache` | Redis | auth-service | Yes | No | Active sessions and tokens |
| `feature_flags_cache` | Redis | feature-flag-service | No | No | Feature flag configurations |

---

## Summary Statistics

| Metric | Count |
|---|---|
| Total data stores | 41 |
| PostgreSQL databases | 27 |
| Redis caches | 7 |
| S3 buckets | 2 |
| MongoDB databases | 1 |
| MongoDB+S3 (hybrid) | 1 |
| Elasticsearch clusters | 1 |
| Other (combined) | 2 |
| Stores containing PII | 19 |
| Stores containing financial data | 17 |
| Stores containing both PII and financial data | 8 |

---

## Governance Notes

- All databases containing PII must comply with LGPD data retention and right-to-erasure requirements.
- Financial databases are subject to Central Bank of Brazil audit requirements and must retain records for a minimum of 5 years.
- Redis caches storing PII must have TTL policies configured to minimize data exposure.
- S3 buckets must have encryption at rest enabled and access logging turned on.
- The `audit_log_store` (Elasticsearch) is append-only and must not allow record deletion outside of regulatory retention policies.
