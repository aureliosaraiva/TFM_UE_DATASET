# Negotiation Engine Service

**svc-41** -- Collections Domain

## Summary

The Negotiation Engine Service generates settlement and renegotiation offers for overdue customers. Given an overdue contract, it evaluates a set of discount rules, produces one or more offer options (e.g., lump-sum with discount, extended term, reduced installment), and tracks offer acceptance or expiry.

It is a pure computation and state-tracking service. It does not communicate with customers -- the dunning service and contact service handle that.

**Runtime:** Kotlin / Spring Boot | **Database:** PostgreSQL (`collections_db`) | **Deployment:** EKS

## Architecture

### Core Pipeline

```
Input: GenerateOffer command (from dunning-service, via SQS)
  |
  v
ContractDataFetcher --> billing-service REST API (get outstanding balance, schedule)
  |
  v
DiscountRuleEvaluator --> evaluates rules from negotiation_rules table
  |
  v
OfferBuilder --> constructs 1-3 offer options
  |
  v
OfferRepository --> persists offers with status PENDING, expiry timestamp
  |
  v
Output: OfferGenerated event (SNS) --> consumed by dunning-service, customer-portal-bff
```

### Discount Rules

Rules are stored in `collections_db.negotiation_rules` and evaluated in priority order. Each rule specifies:
- **Eligibility criteria:** days-past-due range, product type, customer segment, outstanding amount range.
- **Discount parameters:** max principal discount %, max interest discount %, max fee waiver %.
- **Term options:** max extension months, minimum installment amount.

Rules are managed via the backoffice and versioned. When a rule changes, existing unexpired offers are not retroactively modified.

### Offer Lifecycle

| Status | Meaning |
|--------|---------|
| `PENDING` | Offer generated, awaiting customer decision |
| `PRESENTED` | Offer shown to customer (portal or agent) |
| `ACCEPTED` | Customer accepted; triggers renegotiation flow |
| `REJECTED` | Customer explicitly declined |
| `EXPIRED` | TTL reached without response (default: 7 days) |
| `SUPERSEDED` | New offer generated, replacing this one |

An accepted offer publishes `NegotiationAccepted`, which the dunning service uses to resolve the campaign, and the billing service uses to adjust the installment schedule.

## Integration Patterns

**Inbound:** Asynchronous `GenerateOffer` commands from dunning-service. Also supports synchronous `POST /v1/offers/generate` for backoffice-initiated offers.

**Outbound REST (synchronous):** Calls billing-service to retrieve the current outstanding balance and installment schedule. This is the only synchronous dependency.

**Outbound events (asynchronous):** `OfferGenerated`, `NegotiationAccepted`, `NegotiationExpired` published to SNS.

**Offer expiry:** A scheduled job runs hourly to transition `PENDING` or `PRESENTED` offers past their expiry timestamp to `EXPIRED` and publishes `NegotiationExpired`.

## ADR

### ADR-041-001: Rule-based discount engine over hardcoded business logic

**Context:** Initial implementation had discount percentages hardcoded per product type. Product managers needed to adjust discounts frequently in response to portfolio performance and regulatory guidance.

**Decision:** Externalize discount rules into a database-driven rule engine. Rules are versioned, auditable, and editable via the backoffice without deployments.

**Consequences:** Product managers can iterate on discount strategies independently of engineering sprints. Rule conflicts are resolved by priority ordering. Adds complexity: rules must be carefully tested to avoid unintended discounts (guardrails enforce maximum discount caps as a safety net).

### ADR-041-002: Generate multiple offer options per request

**Context:** Behavioral research suggests customers are more likely to engage with a negotiation when presented with choices rather than a single take-it-or-leave-it offer.

**Decision:** Generate up to 3 offer options per request: a lump-sum settlement (highest discount), a short-term plan (moderate discount), and an extended plan (lowest discount, lowest monthly payment).

**Consequences:** Higher negotiation acceptance rates observed in A/B testing (~12% lift). Increases the offer table size by 3x. The customer portal and backoffice UIs were designed to present all options side-by-side.
