# CredityFlow Teams

This document describes the 15 engineering teams that own and operate the CredityFlow platform. Each team is responsible for a bounded domain, its microservices, and the business outcomes within that domain.

---

## 1. team-onboarding

| Field | Value |
|---|---|
| **Team ID** | `team-onboarding` |
| **Team Name** | Onboarding & Proposals |
| **Mission** | Own the customer acquisition funnel from lead to proposal |
| **Team Size** | 8 |
| **Tech Lead** | Rafael Monteiro |
| **Product Owner** | Juliana Ferraz |

**Owned Services:**
- `lead-intake-service`
- `eligibility-service`
- `customer-service`
- `proposal-service`

**KPIs:**
- Lead conversion rate
- Proposal completion rate
- Time-to-proposal

---

## 2. team-documents

| Field | Value |
|---|---|
| **Team ID** | `team-documents` |
| **Team Name** | Document Intelligence |
| **Mission** | Manage document lifecycle from upload to validated storage |
| **Team Size** | 7 |
| **Tech Lead** | Camila Rocha |
| **Product Owner** | Eduardo Pinto |

**Owned Services:**
- `document-upload-service`
- `ocr-worker`
- `document-classifier-service`
- `document-validation-service`
- `document-store-service`

**KPIs:**
- OCR accuracy
- Document processing time
- Validation pass rate

---

## 3. team-identity

| Field | Value |
|---|---|
| **Team ID** | `team-identity` |
| **Team Name** | Identity & Biometrics |
| **Mission** | Verify customer identity through biometric validation |
| **Team Size** | 5 |
| **Tech Lead** | Thiago Nakamura |
| **Product Owner** | Fernanda Lima |

**Owned Services:**
- `biometrics-capture-service`
- `liveness-worker`
- `face-match-service`

**KPIs:**
- Biometric pass rate
- False rejection rate
- Liveness detection accuracy

---

## 4. team-fraud

| Field | Value |
|---|---|
| **Team ID** | `team-fraud` |
| **Team Name** | Fraud Prevention |
| **Mission** | Detect and prevent fraudulent applications |
| **Team Size** | 8 |
| **Tech Lead** | Lucas Andrade |
| **Product Owner** | Mariana Dias |

**Owned Services:**
- `device-fingerprint-service`
- `watchlist-service`
- `fraud-bureau-integration`
- `fraud-scoring-service`
- `fraud-decision-service`

**KPIs:**
- Fraud detection rate
- False positive rate
- Review queue time

---

## 5. team-credit

| Field | Value |
|---|---|
| **Team ID** | `team-credit` |
| **Team Name** | Credit Analysis |
| **Mission** | Assess creditworthiness and make lending decisions |
| **Team Size** | 10 |
| **Tech Lead** | Renata Souza |
| **Product Owner** | Gabriel Teixeira |

**Owned Services:**
- `credit-bureau-integration`
- `income-verification-service`
- `credit-scoring-engine`
- `credit-policy-service`
- `credit-decision-service`

**KPIs:**
- Approval rate
- Default rate
- Decision time
- Policy override rate

---

## 6. team-appraisal

| Field | Value |
|---|---|
| **Team ID** | `team-appraisal` |
| **Team Name** | Collateral Valuation |
| **Mission** | Value collateral assets for secured lending |
| **Team Size** | 6 |
| **Tech Lead** | Andre Carvalho |
| **Product Owner** | Patricia Moreira |

**Owned Services:**
- `appraisal-orchestrator-service`
- `vehicle-appraisal-service`
- `property-appraisal-service`
- `appraisal-report-service`

**KPIs:**
- Appraisal turnaround time
- Valuation accuracy
- Inspection completion rate

---

## 7. team-contracts

| Field | Value |
|---|---|
| **Team ID** | `team-contracts` |
| **Team Name** | Contracts & Legal |
| **Mission** | Generate, validate, and manage loan contracts |
| **Team Size** | 6 |
| **Tech Lead** | Diego Ribeiro |
| **Product Owner** | Larissa Costa |

**Owned Services:**
- `contract-generation-service`
- `compliance-check-service`
- `digital-signature-service`
- `contract-store-service`

**KPIs:**
- Contract generation time
- Compliance pass rate
- Signature completion rate

---

## 8. team-registry

| Field | Value |
|---|---|
| **Team ID** | `team-registry` |
| **Team Name** | Registry & Liens |
| **Mission** | Register liens with government agencies |
| **Team Size** | 5 |
| **Tech Lead** | Bruno Mendes |
| **Product Owner** | Isabela Almeida |

**Owned Services:**
- `registry-orchestrator-service`
- `detran-integration-service`
- `cartorio-integration-service`
- `registry-tracker-service`

**KPIs:**
- Registry success rate
- Registration time
- Retry rate

---

## 9. team-disbursement

| Field | Value |
|---|---|
| **Team ID** | `team-disbursement` |
| **Team Name** | Disbursement & Payments |
| **Mission** | Execute fund transfers and confirm disbursements |
| **Team Size** | 5 |
| **Tech Lead** | Marcos Oliveira |
| **Product Owner** | Aline Santos |

**Owned Services:**
- `disbursement-service`
- `bank-transfer-integration`
- `payment-confirmation-service`

**KPIs:**
- Disbursement success rate
- Transfer time
- Reconciliation accuracy

---

## 10. team-collections

| Field | Value |
|---|---|
| **Team ID** | `team-collections` |
| **Team Name** | Collections & Recovery |
| **Mission** | Manage overdue accounts, dunning, and renegotiation |
| **Team Size** | 12 |
| **Tech Lead** | Vanessa Barros |
| **Product Owner** | Roberto Campos |

**Owned Services:**
- `billing-service`
- `dunning-service`
- `collections-contact-service`
- `negotiation-engine-service`
- `legal-collections-service`
- `renegotiation-eligibility-service`
- `renegotiation-simulation-service`

**KPIs:**
- Recovery rate
- Days past due
- Renegotiation conversion
- Legal action rate

---

## 11. team-portal

| Field | Value |
|---|---|
| **Team ID** | `team-portal` |
| **Team Name** | Customer Experience |
| **Mission** | Provide self-service interfaces for customers |
| **Team Size** | 6 |
| **Tech Lead** | Felipe Azevedo |
| **Product Owner** | Carolina Vieira |

**Owned Services:**
- `customer-portal-bff`

**KPIs:**
- Portal adoption rate
- Self-service resolution rate
- NPS

---

## 12. team-backoffice

| Field | Value |
|---|---|
| **Team ID** | `team-backoffice` |
| **Team Name** | Operations & Backoffice |
| **Mission** | Enable internal operations and manual review workflows |
| **Team Size** | 7 |
| **Tech Lead** | Gustavo Pereira |
| **Product Owner** | Daniela Fonseca |

**Owned Services:**
- `backoffice-bff`
- `case-management-service`

**KPIs:**
- Case resolution time
- Operator throughput
- Queue depth

---

## 13. team-notifications

| Field | Value |
|---|---|
| **Team ID** | `team-notifications` |
| **Team Name** | Communications |
| **Mission** | Manage multi-channel customer and internal notifications |
| **Team Size** | 4 |
| **Tech Lead** | Amanda Duarte |
| **Product Owner** | Pedro Martins |

**Owned Services:**
- `notification-service`
- `notification-template-service`

**KPIs:**
- Delivery rate
- Open rate
- Template usage

---

## 14. team-data

| Field | Value |
|---|---|
| **Team ID** | `team-data` |
| **Team Name** | Data & Analytics |
| **Mission** | Provide data infrastructure, pipelines, and analytics |
| **Team Size** | 6 |
| **Tech Lead** | Ricardo Nunes |
| **Product Owner** | Beatriz Araujo |

**Owned Services:**
- _(No application services currently; manages data pipelines and warehousing externally)_

**KPIs:**
- Data freshness
- Pipeline reliability
- Dashboard adoption

---

## 15. team-platform

| Field | Value |
|---|---|
| **Team ID** | `team-platform` |
| **Team Name** | Platform Engineering |
| **Mission** | Provide shared infrastructure, auth, observability, and developer tools |
| **Team Size** | 8 |
| **Tech Lead** | Leonardo Gomes |
| **Product Owner** | Tatiana Rezende |

**Owned Services:**
- `audit-service`
- `auth-service`
- `feature-flag-service`

**KPIs:**
- Platform uptime
- Deployment frequency
- Incident MTTR
