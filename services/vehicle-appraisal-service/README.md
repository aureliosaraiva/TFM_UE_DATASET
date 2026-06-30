# Vehicle Appraisal Service

**svc-24** | Appraisal Domain | team-appraisal

## Overview

The Vehicle Appraisal Service determines the market value of vehicles offered as collateral for secured loans. It fetches base pricing from Brazil's FIPE table (Tabela FIPE) — the nationally recognised vehicle pricing reference — and applies adjustments for mileage, age, condition, and regional factors to arrive at a fair market valuation.

Accurate vehicle valuation is critical for LTV (Loan-to-Value) calculations. Over-valuation exposes the lender to losses if the loan defaults and the vehicle must be repossessed and sold. Under-valuation unnecessarily restricts borrowers.

The service processes appraisal requests asynchronously via Kafka events and stores all valuation data in PostgreSQL for audit and analytics.

## Getting Started

**Stack:** Kotlin 1.9, Spring Boot 3.x, PostgreSQL 15, Kafka

```bash
# Start dependencies
docker-compose up -d postgres kafka

# Set environment
export DATABASE_URL=jdbc:postgresql://localhost:5432/appraisal_db
export DATABASE_USERNAME=appraisal_svc
export DATABASE_PASSWORD=changeme
export KAFKA_BOOTSTRAP_SERVERS=localhost:9092
export FIPE_API_BASE_URL=https://api.fipe.org.br/v1

# Build and run
./gradlew bootRun

# Verify
curl http://localhost:8080/actuator/health
```

### Running Tests

```bash
./gradlew test                    # Unit tests (FIPE API is mocked)
./gradlew integrationTest         # Integration tests
```

Note: Integration tests use WireMock to simulate the FIPE API. No real external calls are made during testing.

## Architecture

The valuation pipeline has three stages:

1. **FIPE Lookup** — Query the FIPE API for the base reference price using make/model/year/fuel
2. **Depreciation Adjustment** — Apply mileage and age curves to the base price
3. **Condition Adjustment** — If physical inspection data is available, adjust for body and mechanical condition

Each stage is an independent component, making it straightforward to modify one adjustment algorithm without affecting others.

```
appraisal_requested (VEHICLE)
        |
        v
  [FIPE Lookup] --> [Depreciation Adj.] --> [Condition Adj.] --> Persist + Publish
                                                                        |
                                                                        v
                                                              vehicle_appraised
```

The FIPE API response is cached locally (in-memory, 24h TTL) since FIPE prices update only monthly.

See [architecture.md](./architecture.md) for the full component diagram and ADRs.

## API Reference

### POST /api/v1/vehicles/appraise

Triggers a vehicle appraisal. Can be called directly (backoffice) or via event processing.

```json
{
  "proposal_id": "uuid",
  "make": "Honda",
  "model": "Civic",
  "year": 2019,
  "fuel_type": "FLEX",
  "plate": "ABC1D23",
  "odometer_km": 45000,
  "inspection": {
    "body_score": 8,
    "mechanical_score": 9,
    "defects": []
  }
}
```

**Response:**
```json
{
  "valuation_id": "uuid",
  "fipe_base_price": 85000.00,
  "mileage_adjustment": -2550.00,
  "condition_adjustment": 0.00,
  "appraised_value": 82450.00,
  "valuation_date": "2026-03-29",
  "fipe_reference_month": "2026-03"
}
```

### GET /api/v1/vehicles/{id}/valuation

Returns the latest valuation for a vehicle appraisal by its ID.

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `FIPE_API_BASE_URL` | FIPE table API base URL | `https://api.fipe.org.br/v1` |
| `FIPE_CACHE_TTL_HOURS` | Local cache TTL for FIPE prices | `24` |
| `DATABASE_URL` | PostgreSQL connection | — |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka brokers | `localhost:9092` |
| `MILEAGE_DEPRECIATION_RATE_PER_KM` | Depreciation per km driven | `0.00003` |
| `MAX_CONDITION_ADJUSTMENT_PCT` | Max condition-based adjustment | `0.15` |

## Deployment

Deployed to EKS in the `appraisal` namespace:

```bash
helm upgrade --install vehicle-appraisal ./charts/vehicle-appraisal-service \
  --namespace appraisal \
  --values values-production.yaml
```

**Resources:** 512Mi / 250m (request). 2-4 replicas via HPA.

## Monitoring

| Dashboard | Key Metrics |
|-----------|-------------|
| Vehicle Appraisal Operations | Volume, latency, FIPE cache hit rate |
| Valuation Analytics | Adjustment distributions, price segments |

**Alerts:**
- `fipe_api_error_rate_high` — FIPE errors > 5% over 5 min
- `appraisal_latency_p95_high` — p95 > 4s
- `valuation_adjustment_outlier` — adjustment > 40% (possible data issue)

## Known Issues

1. **FIPE coverage gaps** — Some vehicle variants (imports, limited editions, heavy trucks) are absent from the FIPE table. When a lookup returns no result, the service falls back to a manual appraisal queue. This affects roughly 3% of requests.

2. **Odometer fraud** — The service trusts the odometer reading provided. It does not cross-reference vehicle history databases. A tampered odometer leads to an inflated valuation. Integration with DENATRAN vehicle history is under evaluation (APPR-401).

3. **Monthly FIPE update lag** — The FIPE table is updated around the 15th of each month. Vehicles appraised early in the month use the previous month's prices. In volatile market conditions, this can produce outdated valuations.
