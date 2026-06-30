# Renegotiation Eligibility Service (svc-43)

**Collections Domain | Kotlin/Spring Boot | PostgreSQL (renegotiation_db)**

## Overview

Before a customer can renegotiate an overdue contract, CredityFlow must determine whether they are eligible. The Renegotiation Eligibility Service answers a single question: **"Can this contract be renegotiated right now, and under what constraints?"**

The answer depends on a combination of factors: the contract's current status, the customer's history with CredityFlow, regulatory restrictions, and business policy. This service encapsulates all of those rules in one place.

## How It Works

The service exposes a single primary endpoint:

```
POST /v1/eligibility/check
{
  "contractId": "CTR-2025-00847",
  "customerId": "CUS-93821",
  "requestedBy": "CUSTOMER_PORTAL"  // or "BACKOFFICE"
}
```

Response:

```json
{
  "eligible": true,
  "constraints": {
    "maxDiscountPct": 15,
    "maxTermExtensionMonths": 12,
    "minDownPaymentPct": 10,
    "allowedProducts": ["EXTENDED_PLAN", "REDUCED_INSTALLMENT"]
  },
  "reasons": [],
  "evaluatedAt": "2025-11-20T14:30:00Z",
  "rulesVersion": "v2.4"
}
```

Or when ineligible:

```json
{
  "eligible": false,
  "constraints": null,
  "reasons": [
    "CONTRACT_ALREADY_RENEGOTIATED_TWICE",
    "LEGAL_PROCEEDINGS_ACTIVE"
  ],
  "evaluatedAt": "2025-11-20T14:30:05Z",
  "rulesVersion": "v2.4"
}
```

## Internal Components

**RuleEngine:** Evaluates an ordered list of rules loaded from `renegotiation_db.eligibility_rules`. Rules are grouped into "blockers" (hard disqualifiers) and "constraints" (soft limits on the renegotiation parameters). Blocker rules are evaluated first; if any fires, the check short-circuits.

**DataCollector:** Gathers inputs needed by the rule engine from upstream services:
- Contract status and history from billing-service.
- Prior renegotiation count from renegotiation_db.
- Active legal cases from legal-collections-service.
- Customer risk tier from the credit decision service.

All calls are made in parallel with a 3-second timeout. If a dependency is unavailable, the service returns `eligible: false` with reason `DEPENDENCY_UNAVAILABLE` -- a conservative fallback.

**AuditLogger:** Every eligibility check is logged with the full input data, rule evaluation trace, and result. This supports compliance review and dispute resolution.

**RulesAdminAPI:** CRUD endpoints for managing eligibility rules, accessible only from the backoffice BFF. Changes are versioned and take effect immediately (no deployment required).

## Integration Patterns

This service is **synchronous-first**. It is called by the customer portal BFF (when a customer explores renegotiation options), the backoffice BFF (when an agent evaluates a case), and the negotiation engine service (before generating offers).

It makes synchronous REST calls to gather data but does not publish events -- it is a query service with no side effects beyond audit logging.

## Data Flow

```
customer-portal-bff  --->  POST /v1/eligibility/check
backoffice-bff       --->         |
negotiation-engine   --->         v
                          +-------------------+
                          | DataCollector      |----> billing-service (REST)
                          |                    |----> legal-collections-service (REST)
                          |                    |----> credit-decision-service (REST)
                          +-------------------+
                                  |
                                  v
                          +-------------------+
                          | RuleEngine        |----> renegotiation_db.eligibility_rules
                          +-------------------+
                                  |
                                  v
                          +-------------------+
                          | AuditLogger       |----> renegotiation_db.eligibility_audit_log
                          +-------------------+
                                  |
                                  v
                            JSON Response
```

## ADR

### ADR-043-001: Conservative fallback when dependencies are unavailable

**Context:** The eligibility check depends on data from multiple services. If one is down, the service could either assume eligibility (optimistic) or deny it (conservative).

**Decision:** Default to `eligible: false` with a clear reason code when any required dependency is unavailable. The customer sees a "try again later" message; the backoffice agent sees which dependency failed.

**Consequences:** No false positives -- a customer will never receive an offer they should not have. Potential for false negatives during partial outages, which could delay renegotiation. Monitoring dashboards track dependency-related denials to distinguish them from legitimate ineligibility.

### ADR-043-002: Rules versioning for auditability

**Context:** Regulators may ask why a specific customer was denied renegotiation. The answer must reference the exact rules in effect at that time, not the current rules.

**Decision:** Version all rule changes. Each eligibility check records the `rulesVersion` it was evaluated against. Historical rule sets are retained indefinitely.

**Consequences:** Full traceability of any eligibility decision. The rule table grows with each change, but the volume is low (rules change weekly at most). Querying "what rules applied to check X?" is a simple join on version.
