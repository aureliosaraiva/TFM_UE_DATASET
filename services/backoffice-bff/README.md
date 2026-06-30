# Backoffice BFF

| Field | Value |
|-------|-------|
| Service ID | svc-46 |
| Domain | Backoffice |
| Team | team-backoffice |
| Runtime | Node.js / Express |
| Data Store | Redis (`backoffice_session_cache`) |

---

## Overview

The Backoffice BFF (Backend-for-Frontend) serves as the API gateway for CredityFlow's internal operations portal. Back-office agents -- credit analysts, fraud reviewers, compliance officers, and collection managers -- rely on this service to access their daily work queues, review flagged cases, and take action on pending items.

Rather than exposing downstream microservice APIs directly to the front-end, this BFF aggregates and reshapes data from multiple services into view-models optimized for the ops portal UI. It handles session management, request fan-out, and response composition.

### Downstream Dependencies

The BFF does not own any business logic. It delegates to:

- **auth-service** -- session validation, user identity
- **case-management-service** -- CRUD for manual review cases
- **proposal-service** -- loan proposal details
- **fraud-decision-service** -- fraud review outcomes and risk signals
- **credit-decision-service** -- credit assessment results

## Getting Started

```bash
# install dependencies
npm install

# start Redis (required for session cache)
docker compose up -d redis

# run in development mode with hot-reload
npm run dev

# run tests
npm test
```

The server listens on port **3046** by default. Set `PORT` in your `.env` to change this.

You will need a running instance of auth-service (or a mock) for authenticated endpoints. In local development, set `AUTH_MOCK_ENABLED=true` to bypass token validation with a hardcoded test user.

## Architecture

The BFF follows a straightforward request-aggregation pattern:

```
  Ops Portal (React SPA)
         |
         v
  +-------------------+
  | Backoffice BFF    |
  | (Express + Redis) |
  +-------------------+
    |    |    |    |
    v    v    v    v
  auth  case  proposal  fraud/credit
  svc   mgmt  svc       decision svcs
```

**Key patterns:**

- **Parallel fan-out.** Dashboard and queue endpoints issue concurrent requests to downstream services using `Promise.all`, with individual timeouts to prevent a single slow service from blocking the entire response.
- **Session caching.** User session data and frequently-accessed reference data (queue configurations, team assignments) are cached in Redis with short TTLs.
- **Circuit breakers.** Each downstream client wraps calls in a circuit breaker (via `opossum`). If a dependency is unhealthy, the BFF returns degraded responses rather than timing out.
- **Request context propagation.** Trace IDs, user identity, and correlation IDs are forwarded to all downstream calls.

## API Reference

All endpoints are prefixed with `/api/v1/backoffice` and require a valid internal-user JWT.

### Queues

```
GET /api/v1/backoffice/queues
```

Returns all review queues the authenticated agent has access to, with item counts. Queues are derived from case-management-service data, grouped by type (fraud, credit, registry, collections).

### Cases

```
GET /api/v1/backoffice/cases
```

Query parameters: `queue`, `status`, `assigned_to`, `priority`, `page`, `size`.

Lists cases across queues. Enriches each case with summary data from proposal-service and the relevant decision service.

### Actions

```
POST /api/v1/backoffice/actions
```

Body: `{ "case_id": "...", "action": "approve|reject|escalate|reassign", "comment": "..." }`

Dispatches an agent action to case-management-service and triggers any necessary side effects (e.g., notifying the customer, updating the proposal status).

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `PORT` | HTTP listen port | `3046` |
| `REDIS_URL` | Redis connection string | `redis://localhost:6379` |
| `SESSION_TTL_SECONDS` | Session cache TTL | `1800` |
| `AUTH_SERVICE_URL` | Auth service base URL | `http://auth-service:8051` |
| `CASE_MGMT_SERVICE_URL` | Case management service base URL | `http://case-management-service:8047` |
| `PROPOSAL_SERVICE_URL` | Proposal service base URL | `http://proposal-service:8010` |
| `FRAUD_DECISION_SERVICE_URL` | Fraud decision service base URL | `http://fraud-decision-service:8030` |
| `CREDIT_DECISION_SERVICE_URL` | Credit decision service base URL | `http://credit-decision-service:8025` |
| `CIRCUIT_BREAKER_THRESHOLD` | Failure % to open circuit | `50` |
| `DOWNSTREAM_TIMEOUT_MS` | Per-request timeout for downstream calls | `5000` |
| `AUTH_MOCK_ENABLED` | Skip token validation in dev | `false` |

## Deployment

Deployed to AWS EKS in the `backoffice` namespace via Helm.

```bash
helm upgrade --install backoffice-bff infra/helm/backoffice-bff \
  --namespace backoffice \
  -f values-production.yaml
```

**Production configuration:**
- 2 replicas minimum, HPA scales to 4 based on CPU
- 256Mi memory request, 512Mi limit
- 200m CPU request, 500m limit
- Liveness: `GET /health`, Readiness: `GET /ready`

The BFF is stateless aside from the Redis session cache, so scaling and redeployment are straightforward.

## Monitoring

**Health endpoints:**
- `GET /health` -- basic liveness
- `GET /ready` -- checks Redis and downstream service connectivity

**Metrics (Prometheus):**
- `backoffice_bff_request_duration_seconds` -- histogram by route
- `backoffice_bff_downstream_calls_total` -- counter by target service and status
- `backoffice_bff_circuit_breaker_state` -- gauge (0=closed, 1=half-open, 2=open)

**Grafana dashboard:** Backoffice / BFF Overview

**Alerts:**
- Response time p95 > 3s
- Circuit breaker open for any downstream service > 2 minutes
- Error rate > 5% over a 5-minute window

## Known Issues

1. **No request-level caching for case details.** Each page load fetches fresh data from all downstream services. For high-traffic queues this generates significant load on case-management-service. A short-lived (5-10s) response cache would help.

2. **Sequential enrichment for large case lists.** While individual downstream calls use parallel fan-out, enriching a list of 50+ cases still issues many parallel requests. A batch endpoint on case-management-service or proposal-service would reduce chattiness.

3. **Auth mock bypasses RBAC.** When `AUTH_MOCK_ENABLED=true`, the test user has all permissions. Be cautious about running integration tests that rely on role-based access control.
