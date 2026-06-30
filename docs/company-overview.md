# CredityFlow — Company Overview

## About

**CredityFlow** is a digital fintech platform specialized in secured lending, offering vehicle-backed and home-backed loan products across origination, servicing, and collections. Founded in 2018, the company operates a fully digital lending pipeline with integrations to banks, credit bureaus, government registries, and third-party appraisal providers.

## Products

| Product ID | Product Name | Description | Collateral Type |
|-----------|-------------|-------------|-----------------|
| PROD-01 | AutoCredit | Vehicle-backed personal loan | Vehicle |
| PROD-02 | HomeEquity | Home equity loan (home-backed) | Real estate |
| PROD-03 | AutoRefi | Vehicle refinancing | Vehicle |
| PROD-04 | HomeRefi | Home equity refinancing | Real estate |
| PROD-05 | TopUp | Additional credit on existing collateral | Vehicle or Real estate |
| PROD-06 | Renegotiation | Debt renegotiation for overdue customers | Vehicle or Real estate |

## Business Domains

| Domain | Description | Criticality |
|--------|-------------|-------------|
| Onboarding | Lead capture, customer registration, eligibility | High |
| Document Management | Document upload, validation, OCR, classification | High |
| Biometrics | Identity verification via facial recognition and liveness | High |
| Credit Analysis | Credit scoring, policy engine, decision | Critical |
| Fraud Prevention | Fraud signals, device fingerprinting, watchlists | Critical |
| Appraisal | Collateral valuation (vehicle/property inspection) | High |
| Contract | Contract generation, digital signature, compliance | Critical |
| Registry | Lien registration with government agencies (DETRAN/cartório) | High |
| Disbursement | Fund transfer to customer bank account | Critical |
| Collections | Overdue management, dunning, renegotiation | High |
| Customer Portal | Self-service portal for customers | Medium |
| Backoffice | Internal tools for operations, review queues, overrides | Medium |
| Notifications | Email, SMS, push, WhatsApp messaging | Medium |
| Analytics | Reporting, dashboards, data pipelines | Medium |
| Platform | Shared infrastructure, auth, config, feature flags | High |

## Architecture Overview

CredityFlow runs a **microservices architecture** deployed on AWS (EKS). Services communicate via:

- **Synchronous**: REST APIs (internal and external), gRPC for performance-critical paths
- **Asynchronous**: Amazon SQS queues and Amazon SNS topics for event-driven workflows
- **Data stores**: PostgreSQL (primary), MongoDB (document store), Redis (cache/sessions), Amazon S3 (file storage), Elasticsearch (search/logs)

API Gateway (Kong) handles external traffic. Internal service mesh uses Istio. Observability via Datadog (metrics, traces, logs). CI/CD through GitHub Actions.

### Key Architectural Decisions
- Event-driven choreography for the main lending pipeline (onboarding → disbursement)
- Saga pattern for distributed transactions (contract + registry + disbursement)
- CQRS in credit analysis domain for separation of scoring writes and reads
- Feature flags via LaunchDarkly for progressive rollouts
- PII data encrypted at rest and in transit; dedicated PII vaults

## Main End-to-End Flows

### 1. Onboarding Flow
**Trigger**: Customer submits lead via website/app or partner channel
**Steps**:
1. Lead captured by `lead-intake-service`
2. Basic eligibility check by `eligibility-service`
3. Customer registration by `customer-service`
4. Proposal created by `proposal-service`
**Communication**: Sync (REST) for lead intake → Async (event) for downstream
**Human steps**: None (fully automated)

### 2. Document Flow
**Trigger**: `proposal_started` event
**Steps**:
1. Document upload via `document-upload-service`
2. OCR extraction by `ocr-worker`
3. Classification by `document-classifier-service`
4. Validation by `document-validation-service`
5. Storage in `document-store-service`
**Communication**: Async (event-driven pipeline)
**Human steps**: Manual review queue for low-confidence OCR results (backoffice)

### 3. Biometrics Flow
**Trigger**: Documents submitted event
**Steps**:
1. Selfie capture via `biometrics-capture-service`
2. Liveness detection by `liveness-worker`
3. Face match against ID document by `face-match-service`
4. Result stored and event emitted
**Communication**: Sync (capture) → Async (processing)
**Human steps**: Manual review for inconclusive face matches

### 4. Fraud Analysis Flow
**Trigger**: `biometrics_validated` event
**Steps**:
1. Device fingerprint check by `device-fingerprint-service`
2. Watchlist screening by `watchlist-service`
3. Bureau fraud signals by `fraud-bureau-integration`
4. Fraud scoring by `fraud-scoring-service`
5. Decision by `fraud-decision-service`
**Communication**: Async (event-driven)
**Human steps**: Manual review for medium-risk fraud flags

### 5. Credit Analysis Flow
**Trigger**: `fraud_cleared` event
**Steps**:
1. Bureau data fetch by `credit-bureau-integration`
2. Income verification by `income-verification-service`
3. Credit scoring by `credit-scoring-engine`
4. Policy rules evaluation by `credit-policy-service`
5. Credit decision by `credit-decision-service`
**Communication**: Async (event-driven), CQRS for scoring
**Human steps**: Manual override for borderline cases (credit analyst)

### 6. Appraisal Flow
**Trigger**: `credit_approved` event
**Steps**:
1. Appraisal request by `appraisal-orchestrator-service`
2. Vehicle: FIPE table lookup + optional inspection via `vehicle-appraisal-service`
3. Property: Inspection scheduling via `property-appraisal-service`
4. Valuation report by `appraisal-report-service`
**Communication**: Mixed (sync for FIPE lookup, async for inspections)
**Human steps**: Property physical inspection (third-party), vehicle inspection for high-value loans

### 7. Contract Flow
**Trigger**: `appraisal_completed` event
**Steps**:
1. Contract generation by `contract-generation-service`
2. Compliance check by `compliance-check-service`
3. Digital signature via `digital-signature-service`
4. Contract storage by `contract-store-service`
**Communication**: Sync (generation) → Async (signature callback)
**Human steps**: Compliance review for high-value contracts

### 8. Registry Flow
**Trigger**: `contract_signed` event
**Steps**:
1. Registry request by `registry-orchestrator-service`
2. Vehicle lien: DETRAN integration via `detran-integration-service`
3. Property lien: Cartório integration via `cartorio-integration-service`
4. Status tracking by `registry-tracker-service`
**Communication**: Async (government systems are slow)
**Human steps**: Manual follow-up for failed registrations

### 9. Disbursement Flow
**Trigger**: `registry_completed` event
**Steps**:
1. Disbursement order creation by `disbursement-service`
2. Bank transfer via `bank-transfer-integration`
3. Confirmation by `payment-confirmation-service`
**Communication**: Sync (bank API) with async retry
**Human steps**: Manual approval for disbursements > R$500,000

### 10. Collections Flow
**Trigger**: `overdue_installment` event (from `billing-service`)
**Steps**:
1. Dunning campaign by `dunning-service`
2. Contact attempts by `collections-contact-service`
3. Negotiation offers by `negotiation-engine-service`
4. Legal proceedings by `legal-collections-service` (if needed)
**Communication**: Async (event-driven campaigns)
**Human steps**: Phone contact, negotiation approval, legal decisions

### 11. Renegotiation Flow
**Trigger**: Customer request or `renegotiation_requested` event
**Steps**:
1. Eligibility check by `renegotiation-eligibility-service`
2. Simulation by `renegotiation-simulation-service`
3. New terms generation → feeds back into Contract Flow
**Communication**: Sync (simulation) → Async (contract)
**Human steps**: Approval for significant term changes

### 12. Backoffice Flow
**Trigger**: Various (manual review queues, internal requests)
**Steps**:
1. Queue management by `backoffice-bff`
2. Case review via `case-management-service`
3. Override/approval actions
4. Audit logging by `audit-service`
**Communication**: Sync (REST APIs)
**Human steps**: All steps are human-operated

## Critical Domains
- **Credit Analysis** and **Fraud Prevention**: Direct financial risk exposure
- **Contract** and **Disbursement**: Legal and financial commitments
- **Registry**: Government compliance requirements

## Sync vs Async Summary
| Pattern | Used In |
|---------|---------|
| Synchronous (REST/gRPC) | Lead intake, eligibility checks, FIPE lookups, contract generation, bank transfers, backoffice operations |
| Asynchronous (Events/Queues) | Document pipeline, biometrics processing, fraud analysis, credit analysis, registry tracking, collections campaigns, notifications |
| Mixed | Appraisal (sync lookup + async inspection), disbursement (sync transfer + async confirmation) |
