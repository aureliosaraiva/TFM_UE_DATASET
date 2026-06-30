# renegotiation-simulation-service

**svc-44** | Collections | team-collections | **Beta**

---

## Overview

This service picks up where the eligibility service leaves off. Once a borrower is confirmed eligible for renegotiation, the simulation service generates concrete repayment scenarios they can choose from: extend the term, reduce the rate, pay a discounted lump sum, or some combination. The borrower (or an operations agent) reviews the options, and the chosen terms flow downstream to contract-generation-service to be formalized.

The service is currently in **beta**. API contracts are stabilizing but may still change. Consumers should handle schema evolution defensively.

## Getting Started

Kotlin/Spring Boot service, built with Gradle. Same local setup pattern as the rest of the collections domain:

```bash
docker-compose up -d postgres kafka
./gradlew bootRun --args='--spring.profiles.active=local'
```

Port: `8085` | Health: `http://localhost:8085/actuator/health`

### Environment Variables

| Variable | Description |
|---|---|
| `SPRING_DATASOURCE_URL` | JDBC URL for renegotiation_db |
| `SPRING_KAFKA_BOOTSTRAP_SERVERS` | Kafka brokers |
| `SIMULATION_SCENARIO_TEMPLATES` | Comma-separated list of default scenario types (default: `EXTENSION,RATE_CUT,LUMP_SUM`) |
| `SIMULATION_PRECOMPUTE_ENABLED` | Whether to pre-compute scenarios on eligibility events (default: `true`) |
| `SIMULATION_CACHE_TTL_MINUTES` | TTL for pre-computed simulation cache (default: 30) |

## Architecture

Two entry points feed into the simulation engine:

1. **Synchronous (API):** The `POST /api/v1/renegotiation/simulate` endpoint accepts a request, runs calculations on the spot, persists the result, and returns it.
2. **Asynchronous (events):** When a `renegotiation_eligible` event arrives, the service pre-computes scenarios using default templates and caches them. Subsequent `GET` requests for that simulation are served from the cache.

The calculation engine is stateless and purely functional -- given a debt snapshot and a scenario template, it produces a `SimulationScenario` with projected installments. This makes it straightforward to test and to scale horizontally.

**Entities:** `RenegotiationSimulation`, `SimulationScenario`, `NewTerms`

**Database:** `renegotiation_db` (PostgreSQL, shared with renegotiation-eligibility-service). Contains financial data.

See [architecture.md](./architecture.md) for diagrams and architectural decision records.

## API Reference

### POST /api/v1/renegotiation/simulate

Generate renegotiation scenarios. Returns multiple options the borrower can choose from.

**Request:**
```json
{
  "borrower_id": "bor-100",
  "proposal_id": "prop-200",
  "scenario_types": ["EXTENSION", "RATE_CUT", "LUMP_SUM"]
}
```

**Response 200:**
```json
{
  "simulation_id": "sim-300",
  "scenarios": [
    {
      "type": "EXTENSION",
      "additional_months": 12,
      "new_monthly_amount": 450.00,
      "total_cost": 15400.00
    },
    {
      "type": "RATE_CUT",
      "new_annual_rate": 8.5,
      "new_monthly_amount": 520.00,
      "total_cost": 14560.00
    },
    {
      "type": "LUMP_SUM",
      "discount_pct": 20.0,
      "settlement_amount": 9600.00,
      "due_date": "2026-04-30"
    }
  ],
  "valid_until": "2026-04-28T23:59:59Z"
}
```

Returns `422` if the borrower is not eligible or the debt state is inconsistent.

### GET /api/v1/renegotiation/simulations/{id}

Retrieve a previously generated simulation. Returns `404` if expired or not found.

## Configuration

Scenario templates control what kinds of offers are generated. The default set covers the three most common strategies (extension, rate cut, lump sum), but additional templates can be registered in the `scenario_templates` config table.

Each template defines:
- Scenario type and display name
- Parameter ranges (e.g., extension: 3-24 additional months)
- Calculation method reference

Application settings of note:
- **`simulation.max-scenarios-per-request`** -- caps the number of scenarios generated (default: 5)
- **`simulation.precompute.concurrency`** -- parallel threads for event-driven pre-computation (default: 4)

## Deployment

```bash
./gradlew bootBuildImage --imageName=credityflow/renegotiation-simulation-service:latest

helm upgrade --install reneg-simulation deploy/helm \
  --namespace collections \
  --values deploy/helm/values-staging.yaml
```

**Beta notice:** This service is tagged with `lifecycle_status: beta` in the service catalog. Production traffic is routed through a feature flag. Ensure the flag is enabled in your target environment before deploying.

Database migrations are managed with Flyway. Coordinate with renegotiation-eligibility-service since they share `renegotiation_db`.

## Monitoring

| Metric | Description |
|---|---|
| `renegotiation_simulations_created_total` | Total simulations generated |
| `renegotiation_scenarios_per_simulation` | Average scenarios per simulation |
| `renegotiation_simulation_duration_seconds` | End-to-end simulation latency |
| `renegotiation_precomputed_cache_hit_rate` | Cache effectiveness for pre-computed results |

**Alerts:**
- *SimulationLatencyHigh* -- P99 over 5 seconds; likely indicates heavy load or a slow dependency.
- *PrecomputationBacklog* -- more than 500 pending pre-computations; scale up consumer threads.

## Known Issues

1. **Beta stability:** API response shapes may evolve. Consumers should use tolerant readers (ignore unknown fields, handle missing optional fields).
2. **Shared database contention:** Under heavy load, the shared `renegotiation_db` can experience lock contention between this service and the eligibility service. Connection pool sizing should account for both services.
3. **Projection accuracy:** Simulated amounts are projections, not binding terms. Final terms are calculated and locked by contract-generation-service. Small discrepancies are possible due to rounding and timing differences.
4. **Pre-computation cache invalidation:** If a borrower makes a payment between pre-computation and retrieval, the cached scenarios become stale. The cache TTL (default 30 minutes) limits the staleness window but does not eliminate it.
