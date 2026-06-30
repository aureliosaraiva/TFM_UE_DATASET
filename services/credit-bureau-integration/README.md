# Credit Bureau Integration

**Service ID:** svc-18 | **Team:** team-credit | **Domain:** Credit Analysis

## Overview

The Credit Bureau Integration service is the gateway between CredityFlow's credit analysis pipeline and external credit bureaus. It fetches credit reports from ServicosBureau — Brazil's primary credit information provider — and makes them available to internal services through an event-driven architecture.

When a loan applicant passes fraud checks, this service retrieves their full credit profile: credit history, outstanding debts, and bureau score. To control costs and reduce latency, responses are cached in Redis with a 24-hour TTL. The service handles all the complexity of interfacing with the external provider — retry logic, circuit breaking, rate-limit management, and data normalisation — so that downstream consumers receive a clean, consistent `BureauReport` model regardless of upstream API quirks.

This is a **PII-sensitive** service. It processes and caches CPF numbers, names, and financial obligation data.

## Getting Started

### Prerequisites

- JDK 17+
- Kotlin 1.9+
- Docker (for local Redis)
- Access to the CredityFlow Kafka cluster (or a local instance)

### Running Locally

```bash
# Start Redis
docker run -d --name bureau-redis -p 6379:6379 redis:7-alpine

# Set required environment variables
export SERVICOS_BUREAU_API_KEY=<your-sandbox-key>
export SERVICOS_BUREAU_BASE_URL=https://sandbox.servicosbureau.com.br
export KAFKA_BOOTSTRAP_SERVERS=localhost:9092
export REDIS_HOST=localhost
export REDIS_PORT=6379

# Build and run
./gradlew bootRun
```

The service will start on port `8080`. You can verify it's running with `GET /actuator/health`.

### Running Tests

```bash
./gradlew test                    # Unit tests
./gradlew integrationTest         # Integration tests (requires Redis + Kafka)
```

## Architecture

The service follows a hexagonal architecture pattern. The domain core defines `BureauReport`, `CreditHistory`, and `DebtRecord` entities, which are independent of the external API contract. An adapter layer translates ServicosBureau responses into the domain model.

**Event flow:**
```
fraud_cleared (Kafka) --> [credit-bureau-integration] --> bureau_data_fetched (Kafka)
```

Cache lookup happens before the external call. If a valid (non-expired) entry exists for the customer's CPF hash, the cached report is used and the external call is skipped entirely. Cache keys are hashed CPFs — never raw PII.

For more details, see [architecture.md](./architecture.md).

## API Reference

### POST /api/v1/credit-bureau/fetch

Triggers a bureau data fetch for a customer/proposal pair. This is an **internal** endpoint protected by service-to-service JWT authentication.

**Request body:**
```json
{
  "proposal_id": "uuid",
  "customer_id": "uuid",
  "cpf": "12345678901"
}
```

**Response (200):**
```json
{
  "report_id": "uuid",
  "customer_id": "uuid",
  "bureau_score": 720,
  "credit_history": [...],
  "debt_records": [...],
  "fetched_at": "2026-03-29T10:00:00Z",
  "source": "CACHE | LIVE"
}
```

**Error codes:** `502` (bureau unavailable), `429` (rate limited), `404` (customer not found at bureau).

## Configuration

| Variable | Description | Default |
|---|---|---|
| `SERVICOS_BUREAU_API_KEY` | API key for ServicosBureau | — (required) |
| `SERVICOS_BUREAU_BASE_URL` | Base URL for the bureau API | — (required) |
| `REDIS_HOST` | Redis hostname | `localhost` |
| `REDIS_PORT` | Redis port | `6379` |
| `REDIS_CACHE_TTL_HOURS` | Cache TTL in hours | `24` |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka broker addresses | `localhost:9092` |
| `BUREAU_CIRCUIT_BREAKER_THRESHOLD` | Failure count before circuit opens | `5` |
| `BUREAU_RETRY_MAX_ATTEMPTS` | Max retry attempts for external calls | `3` |

## Deployment

Deployed as a container on AWS EKS via Helm chart. The CI/CD pipeline runs on GitHub Actions.

```bash
# Deploy to staging
helm upgrade --install credit-bureau-integration ./charts/credit-bureau-integration \
  --namespace credit-analysis \
  --values values-staging.yaml

# Verify rollout
kubectl rollout status deployment/credit-bureau-integration -n credit-analysis
```

**Resource requirements:** 512 Mi memory, 250m CPU (request); 1 Gi memory, 500m CPU (limit). Redis is provisioned as an AWS ElastiCache cluster shared with other credit-analysis services.

## Monitoring

- **Grafana dashboard:** "Credit Bureau Operations" — cache hit ratio, call volume, latency percentiles
- **Key metrics:** `bureau_fetch_total`, `bureau_fetch_duration_seconds`, `bureau_cache_hit_ratio`
- **Alerts:**
  - `bureau_error_rate_high` — external error rate > 5% over 5 min
  - `bureau_latency_p99_high` — p99 latency > 3s
  - `bureau_cache_eviction_spike` — eviction rate doubles within 10 min

Logs are shipped to CloudWatch and indexed in OpenSearch. Structured JSON format with trace IDs for correlation.

## Known Issues

1. **Stale cache on bureau data corrections:** If ServicosBureau corrects a record within the 24h TTL window, the cached version will be stale until it expires. There is no cache invalidation webhook from the bureau. Mitigation: the POST endpoint accepts a `force_refresh=true` parameter to bypass cache.

2. **Rate limit sharing:** The ServicosBureau API key is shared across environments. High traffic in staging can consume production rate-limit quota. Tracked in JIRA CRED-1847.

3. **Redis failover latency:** During ElastiCache failover, there is a 10-20s window where cache reads fail. The service falls back to live bureau calls during this period, which can spike external API usage.
