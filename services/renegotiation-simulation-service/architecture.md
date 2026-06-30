# Renegotiation Simulation Service

> **svc-44** | Collections | Kotlin/Spring Boot | PostgreSQL (renegotiation_db)
>
> **Lifecycle: BETA** -- This service is under active development. APIs may change without prior notice.

---

## Purpose

The Renegotiation Simulation Service allows customers and backoffice agents to explore "what-if" scenarios for restructuring overdue debt. Given a contract and a set of renegotiation parameters (new term, down payment, discount), it computes the projected new schedule -- monthly payments, total cost, interest savings -- without committing any changes.

Simulations are ephemeral by design. They are cached briefly for UX responsiveness but carry no contractual weight until a customer formally accepts an offer through the negotiation engine.

## Architecture

### Request/Response Model

```
POST /v1/simulations
{
  "contractId": "CTR-2025-00847",
  "scenarios": [
    { "termMonths": 12, "downPaymentPct": 20, "interestRateOverride": null },
    { "termMonths": 24, "downPaymentPct": 10, "interestRateOverride": null },
    { "termMonths": 6,  "downPaymentPct": 50, "interestRateOverride": null }
  ]
}
```

The service returns a simulation result for each scenario, including the projected installment schedule, total cost comparison against the original contract, and a savings breakdown.

### Components

| Component | Role |
|-----------|------|
| `SimulationController` | REST API. Validates input, enforces rate limits (10 simulations/min per customer). |
| `ContractSnapshotFetcher` | Retrieves current contract state from billing-service. Captures outstanding principal, accrued interest, fees, and original terms. |
| `EligibilityGate` | Calls renegotiation-eligibility-service to verify the contract can be renegotiated and obtains constraint boundaries. Rejects simulations that exceed allowed parameters. |
| `AmortizationCalculator` | Pure computation module. Supports SAC (constant amortization) and Price (constant payment) methods. Calculates IOF tax impact on the renegotiated amount. |
| `SimulationCache` | Stores simulation results in PostgreSQL with a 30-minute TTL. Enables the customer portal to retrieve results without re-computation when the user navigates back. |

### What "Beta" Means for This Service

- The simulation accuracy is validated against the billing service's own calculations, but edge cases (partial prepayments, variable-rate contracts) are still being refined.
- The API contract (request/response schema) may evolve. Consumers should handle unknown fields gracefully.
- Feature-flagged via the feature-flag-service (`renegotiation-simulation-enabled`). Currently enabled for 100% of backoffice users and 30% of customer portal users.

## Data Flow

```
Customer Portal / Backoffice BFF
        |
        | POST /v1/simulations
        v
SimulationController
        |
        +---> EligibilityGate ---> renegotiation-eligibility-service (REST)
        |
        +---> ContractSnapshotFetcher ---> billing-service (REST)
        |
        v
AmortizationCalculator (pure computation, no I/O)
        |
        v
SimulationCache ---> renegotiation_db.simulations (TTL: 30 min)
        |
        v
JSON Response (projected schedules per scenario)
```

This service is entirely synchronous and read-only. It does not publish events or modify any contract state.

## Integration Patterns

- **Synchronous dependencies:** billing-service (contract data), renegotiation-eligibility-service (constraints). Both are required -- if either is unavailable, the simulation request fails with a 503.
- **No event production:** This is a stateless query service (apart from the ephemeral cache). It has no downstream consumers.
- **Rate limiting:** Applied at the controller level using a token-bucket algorithm per customer ID. Prevents abuse from the self-service portal.
- **Feature flag:** The service checks `renegotiation-simulation-enabled` at the BFF level. The service itself always accepts requests; gating happens upstream.

## ADR

### ADR-044-001: Ephemeral simulation cache over persistent storage

**Context:** Early design persisted all simulations permanently for analytics. This generated significant write volume (customers often run 5-10 simulations before deciding) with low analytical value.

**Decision:** Store simulations with a 30-minute TTL. If analytics on simulation patterns is needed later, the service will publish anonymized events to a data lake rather than retaining individual records.

**Consequences:** Reduced database storage and write load by ~90%. Simulations cannot be referenced after 30 minutes -- if the customer returns later, they re-simulate. This is acceptable because contract state may have changed in the interim.

### ADR-044-002: Launch as beta with feature-flag gating

**Context:** The amortization calculator needed validation against production contract data. A full launch risked showing customers inaccurate projections, damaging trust.

**Decision:** Launch in beta behind a feature flag. Start with backoffice-only access (agents can verify accuracy), then gradually roll out to the customer portal.

**Consequences:** Controlled blast radius. Backoffice agents provide feedback on edge cases before customers see them. The beta label sets expectations with internal stakeholders. Requires coordination with the feature-flag-service for rollout percentage changes.
