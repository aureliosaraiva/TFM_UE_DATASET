# Watchlist Service

## Overview

The watchlist-service is responsible for screening customers against sanctions lists, PEP (Politically Exposed Persons) lists, and CredityFlow's internal watchlists. It sits within the Fraud Prevention domain and is a critical compliance checkpoint in the onboarding pipeline.

When a customer's biometrics are validated, the fraud pipeline triggers a screening request. The service fetches the customer's identity data from customer-service, runs fuzzy matching against all configured watchlist sources, and publishes a `watchlist_checked` event with the results.

**Owner:** team-fraud
**Stack:** Kotlin / Spring Boot / PostgreSQL

## Getting Started

### Prerequisites

- JDK 17+
- Docker & Docker Compose (for local PostgreSQL)
- Access to the `customer-service` instance (or mock)

### Running Locally

```bash
# Start dependencies
docker-compose up -d postgres

# Run the service
./gradlew bootRun --args='--spring.profiles.active=local'
```

The service starts on port `8094` by default. Swagger UI is available at `http://localhost:8094/swagger-ui.html`.

### Running Tests

```bash
./gradlew test
```

Integration tests use Testcontainers for PostgreSQL. Make sure Docker is running.

### Seeding Watchlist Data

For local development, a seed script loads a small test watchlist:

```bash
./gradlew seedWatchlists
```

This populates the database with ~500 synthetic entries across OFAC, UN, and internal sources. **Never use production watchlist data in local environments.**

## Architecture

The service follows a straightforward layered architecture:

- **Controller layer** exposes REST endpoints for synchronous screening and result retrieval.
- **Service layer** orchestrates the screening flow: fetch identity, match against sources, aggregate results.
- **Matching engine** uses a combination of Jaro-Winkler similarity and phonetic encoding (Double Metaphone) for fuzzy name matching. The threshold is configurable per source.
- **Repository layer** handles persistence of screening results and watchlist entries.

Watchlist sources are updated by a scheduled job (`WatchlistSourceUpdater`) that runs daily for sanctions lists and weekly for PEP lists. The updater downloads source files, parses them, normalizes entries, and upserts into the database.

### Important Implementation Details

The fuzzy matching threshold was originally set at 0.85 but was lowered to 0.78 after an incident where a sanctioned individual with a transliterated name variant was not caught. This increased false positives by roughly 15%, but the compliance team accepted the tradeoff. The threshold is configured via `watchlist.matching.similarity-threshold` in application.yml.

The matching engine processes sources in parallel using coroutines. If any single source times out (default 5s), the screening still completes but the result is marked as `PARTIAL` and an alert fires. This was a deliberate design choice to avoid blocking the entire fraud pipeline due to one slow source.

## API Reference

### POST /api/v1/watchlist/screen

Submit a screening request. The service fetches customer data from customer-service using the provided `customerId`.

**Request:**
```json
{
  "customerId": "uuid",
  "proposalId": "uuid",
  "screeningType": "FULL"
}
```

**Response (200):**
```json
{
  "screeningId": "uuid",
  "status": "COMPLETED",
  "matchCount": 0,
  "highestConfidence": null,
  "matches": [],
  "sourcesScreened": ["OFAC_SDN", "UN_CONSOLIDATED", "COAF_PEP", "INTERNAL"],
  "screenedAt": "2025-01-15T10:30:00Z"
}
```

### GET /api/v1/watchlist/results/{id}

Retrieve screening results by screening ID.

**Response (200):** Same structure as screening response above.

## Configuration

Key configuration properties in `application.yml`:

| Property | Default | Description |
|---|---|---|
| `watchlist.matching.similarity-threshold` | `0.78` | Minimum Jaro-Winkler similarity for a match |
| `watchlist.matching.phonetic-enabled` | `true` | Enable phonetic matching as secondary strategy |
| `watchlist.sources.ofac.update-cron` | `0 2 * * *` | Cron for OFAC source updates |
| `watchlist.sources.pep.update-cron` | `0 3 * * 1` | Cron for PEP source updates (weekly) |
| `watchlist.screening.source-timeout-ms` | `5000` | Per-source matching timeout |
| `customer-service.base-url` | `http://customer-service:8080` | Customer service URL |

Environment variables for secrets:

- `WATCHLIST_DB_PASSWORD` - PostgreSQL password
- `KAFKA_BOOTSTRAP_SERVERS` - Kafka broker addresses

## Deployment

Deployed as a Kubernetes Deployment in the `fraud` namespace. The service requires:

- PostgreSQL connection to `watchlist_db`
- Kafka connectivity for event consumption and publishing
- Network access to `customer-service`
- Sufficient memory for in-memory watchlist index (~512MB for full production dataset)

The watchlist index is loaded into memory on startup for fast matching. This means the pod needs time to warm up (about 30-45 seconds). The readiness probe is configured with an initial delay of 60 seconds to account for this.

**Scaling notes:** The service is stateless from a request perspective (the in-memory index is read-only after load). It can be horizontally scaled, but each pod loads the full index, so memory usage scales linearly with replicas.

## Monitoring

### Key Dashboards

- **Watchlist Screening Overview** - screening volume, match rates, latency percentiles
- **Source Freshness Monitor** - age of each watchlist source, update success/failure

### Alerts

| Alert | Condition | Severity |
|---|---|---|
| High Screening Latency | p99 > 3s for 5 minutes | Warning |
| Source Stale | Any source not updated for 48+ hours | Critical |
| Screening Errors | Error rate > 2% for 5 minutes | Critical |
| Partial Screening | More than 10 partial results in 1 hour | Warning |

## Known Issues

- **Memory pressure on startup:** Loading the full watchlist index takes ~30s and allocates significant heap. If the pod is OOM-killed during startup, increase the memory limit. Current production setting is 1.5Gi.
- **COAF PEP list format changes:** The COAF occasionally changes their PEP list export format without notice. The parser has been made resilient to minor variations, but major format changes require a code update. Last incident was March 2025.
- **False positive rate:** With the current similarity threshold of 0.78, approximately 3-5% of screenings produce at least one match that requires manual review. The compliance team reviews these via the backoffice.
- **No real-time source updates:** Watchlist sources are updated on a schedule, not in real-time. There is a theoretical window where a newly sanctioned individual could pass screening. This is an accepted risk documented in the compliance risk register.
