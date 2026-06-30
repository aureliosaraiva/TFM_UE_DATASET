# Property Appraisal Service

`svc-25` -- Appraisal -- team-appraisal -- Kotlin/Spring Boot

---

## Overview

Real estate appraisal is inherently more complex and slower than vehicle valuation. Properties are unique, local markets vary wildly, and regulatory requirements mandate physical inspections for home-equity loans above certain thresholds.

This service handles the full property valuation workflow: automated market comparable analysis using internal transaction data, physical inspection coordination through the InspecaoExpress partner, and final valuation computation that blends both data sources. The result feeds into LTV calculations for home-equity and property-backed lending products.

Unlike the vehicle appraisal service (which completes in seconds), property appraisals can take days due to the physical inspection requirement. The service is designed around this asynchronous reality, with webhook-based inspection callbacks and status tracking throughout.

## Getting Started

### Prerequisites

- JDK 17+, Kotlin 1.9+
- PostgreSQL 15+
- Kafka
- InspecaoExpress sandbox credentials (for integration testing)

### Local Development

```bash
docker-compose up -d postgres kafka

export DATABASE_URL=jdbc:postgresql://localhost:5432/appraisal_db
export DATABASE_USERNAME=appraisal_svc
export DATABASE_PASSWORD=changeme
export KAFKA_BOOTSTRAP_SERVERS=localhost:9092
export INSPECAO_EXPRESS_API_KEY=sandbox-key
export INSPECAO_EXPRESS_BASE_URL=https://sandbox.inspecaoexpress.com.br/api

./gradlew bootRun
```

### Tests

```bash
./gradlew test                    # Unit tests
./gradlew integrationTest         # Integration tests (WireMock for InspecaoExpress)
```

## Architecture

The property appraisal process has two parallel tracks that converge:

```
appraisal_requested (PROPERTY)
        |
        +---> [Market Comparable Analysis]  ---+
        |                                      +--> [Valuation Merge] --> property_appraised
        +---> [Inspection via InspecaoExpress] -+
```

**Market Comparable Analysis** runs immediately: queries the internal property database for recent sales within a configurable radius, computes a weighted average based on similarity scores (area, rooms, construction quality, neighbourhood).

**Physical Inspection** is scheduled asynchronously through the InspecaoExpress API. The partner dispatches an inspector, who submits findings via their platform. InspecaoExpress sends a webhook callback to this service when the inspection report is ready.

Once both tracks complete, the service merges the data: market analysis provides the base value, inspection results apply condition adjustments (positive or negative). The confidence level reflects comparable coverage and inspection completeness.

Detailed architecture and ADRs are in [architecture.md](./architecture.md).

## API Reference

### POST /api/v1/properties/appraise

Initiates a property appraisal.

```json
{
  "proposal_id": "uuid",
  "address": {
    "street": "Rua das Flores, 123",
    "neighbourhood": "Jardim Paulista",
    "city": "Sao Paulo",
    "state": "SP",
    "zip": "01401-000"
  },
  "property_type": "APARTMENT",
  "area_sqm": 72,
  "rooms": 2,
  "parking_spaces": 1,
  "construction_year": 2010
}
```

**Response (202 Accepted):**
```json
{
  "appraisal_id": "uuid",
  "status": "IN_PROGRESS",
  "estimated_completion": "2026-04-02T00:00:00Z"
}
```

### GET /api/v1/properties/{id}/valuation

Returns the current valuation status and results. Status will be `IN_PROGRESS` until both market analysis and inspection are complete.

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection | — |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka brokers | `localhost:9092` |
| `INSPECAO_EXPRESS_API_KEY` | API key for InspecaoExpress | — |
| `INSPECAO_EXPRESS_BASE_URL` | InspecaoExpress API URL | — |
| `COMPARABLE_SEARCH_RADIUS_KM` | Radius for comparable property search | `2.0` |
| `MIN_COMPARABLES_REQUIRED` | Minimum comparables for confident valuation | `3` |
| `INSPECTION_WEBHOOK_PATH` | Path for InspecaoExpress callbacks | `/webhooks/inspection` |

## Deployment

```bash
helm upgrade --install property-appraisal ./charts/property-appraisal-service \
  --namespace appraisal \
  --values values-production.yaml
```

**Important:** The webhook endpoint must be reachable from InspecaoExpress's servers. In production, this is exposed through an API Gateway with IP whitelisting and signature verification.

**Resources:** 512Mi / 250m (request). 2 replicas minimum. Scaling is less critical than for vehicle appraisals since inspection latency dominates.

## Monitoring

- **Dashboard:** "Property Appraisal Pipeline" — volume, status funnel, avg completion time (hours/days)
- **Dashboard:** "Inspection Tracking" — scheduling success rate, partner SLA adherence, callback latency
- **Dashboard:** "Market Analysis Quality" — comparable coverage per region, confidence distribution

**Key alerts:**
- `inspection_scheduling_failure` — scheduling errors > 3% over 1 hour
- `appraisal_completion_delay` — average completion > 72 hours
- `low_comparable_coverage` — < 3 comparables for > 20% of valuations

## Known Issues

1. **Rural property coverage** — Properties in rural areas or small towns often have insufficient comparable transactions. The service falls back to a wider search radius, but valuations in these areas have lower confidence and may require manual review. Approximately 12% of property appraisals are affected.

2. **InspecaoExpress webhook reliability** — Webhook delivery from InspecaoExpress is not guaranteed. The service runs a reconciliation job every 6 hours that polls for completed inspections that were not received via webhook. This introduces additional delay for affected appraisals.

3. **Inspection scheduling delays** — InspecaoExpress has limited inspector availability in certain regions. Scheduling can take 3-7 business days in underserved areas. There is no visibility into their inspector capacity ahead of time, making it difficult to set accurate completion estimates for customers.

4. **Market data freshness** — The internal comparable database depends on notary records, which can be 30-90 days behind actual transactions. Fast-moving markets may show stale comparable prices.
