# Customer Portal BFF

**svc-45** | Customer Portal | team-portal

## Overview

This is the Backend-for-Frontend that powers CredityFlow's customer-facing self-service portal. Customers use the portal to track their loan proposals, upload documents, view billing statements, and manage payments. The BFF sits between the React-based portal SPA and the constellation of backend microservices, aggregating data into optimized view-models and shielding the front-end from internal service complexity.

Built with **Node.js and Express**, it maintains session state in **Redis** (`portal_session_cache`) and communicates with four core backend services:

- **auth-service** for customer authentication and token validation
- **proposal-service** for loan proposal status and details
- **customer-service** for customer profile data
- **billing-service** for invoices, payment schedules, and payment history

The BFF is intentionally thin. It contains no business logic -- only orchestration, transformation, and caching.

## Getting Started

### Quick Start

```bash
npm install
cp .env.example .env        # configure local service URLs
docker compose up -d redis
npm run dev                  # starts on port 3045 with nodemon
```

### Test Suite

```bash
npm test                     # jest, ~90 tests
npm run test:integration     # requires downstream services or mocks
```

For local development without running all dependencies, the service supports a mock mode:

```bash
MOCK_DOWNSTREAM=true npm run dev
```

This returns fixture data for all downstream calls, useful for front-end development.

## Architecture

```
  Customer Browser (React SPA)
           |
           | HTTPS
           v
  +---------------------+
  | Customer Portal BFF |──── Redis (session cache)
  +---------------------+
      |       |       |       |
      v       v       v       v
    auth    proposal  customer  billing
    svc     svc       svc       svc
```

### Design Principles

1. **View-model driven.** Each endpoint returns exactly the data the corresponding UI view needs -- no more, no less. This minimizes payload size and simplifies the front-end.

2. **Graceful degradation.** If billing-service is down, the dashboard still loads; the payments section shows a "temporarily unavailable" placeholder. Each downstream call is independent and wrapped in error handling.

3. **Edge caching.** Responses for relatively stable data (customer profile, active proposal details) are cached in Redis for 60 seconds. Volatile data (payment status) is always fetched fresh.

4. **Security boundary.** The BFF validates the customer JWT on every request and ensures customers can only access their own data. It acts as an authorization enforcement point.

## API Reference

All endpoints are under `/api/v1/portal` and require a valid customer JWT.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/dashboard` | Aggregated overview: active proposals, upcoming payments, recent activity. Calls proposal-service, billing-service, and customer-service in parallel. |
| `GET` | `/proposals` | List of the customer's loan proposals with current status. |
| `GET` | `/documents` | Documents associated with the customer's proposals (IDs, types, upload status). Sourced from proposal-service. |
| `GET` | `/payments` | Payment schedule and history from billing-service. |

### Response Format

All responses follow the standard CredityFlow envelope:

```json
{
  "data": { ... },
  "meta": { "timestamp": "...", "request_id": "..." }
}
```

Errors use `{ "error": { "code": "...", "message": "..." } }`.

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `PORT` | Server port | `3045` |
| `REDIS_URL` | Redis connection URL | `redis://localhost:6379` |
| `SESSION_SECRET` | Express session signing secret | -- |
| `AUTH_SERVICE_URL` | Auth service URL | `http://auth-service:8051` |
| `PROPOSAL_SERVICE_URL` | Proposal service URL | `http://proposal-service:8010` |
| `CUSTOMER_SERVICE_URL` | Customer service URL | `http://customer-service:8005` |
| `BILLING_SERVICE_URL` | Billing service URL | `http://billing-service:8040` |
| `CACHE_TTL_SECONDS` | Default Redis cache TTL | `60` |
| `DOWNSTREAM_TIMEOUT_MS` | Timeout for downstream HTTP calls | `3000` |
| `MOCK_DOWNSTREAM` | Return fixture data instead of calling services | `false` |

## Deployment

The service runs on **AWS EKS** in the `portal` namespace.

```bash
helm upgrade --install customer-portal-bff infra/helm/customer-portal-bff \
  --namespace portal -f values-production.yaml
```

Sizing:
- 2-4 replicas (HPA on CPU, target 60%)
- 128Mi-256Mi memory
- 100m-300m CPU
- Liveness probe: `GET /health`
- Readiness probe: `GET /ready`

The service is stateless (Redis is a cache, not a persistence layer). Pods can be freely recycled.

## Monitoring

Dashboards live in **Grafana > Customer Portal > BFF**.

Key signals:

- `portal_bff_http_requests_total` -- request count by route and status code
- `portal_bff_response_time_seconds` -- histogram by route
- `portal_bff_downstream_errors_total` -- errors by target service
- `portal_bff_cache_hit_ratio` -- Redis cache effectiveness

Alerts:
- Error rate on `/dashboard` > 3% for 5 minutes
- p95 latency on any route > 2 seconds
- Redis connection failures

## Known Issues

1. **No WebSocket support.** The portal uses polling (every 30s) to check for proposal status updates. Real-time push via WebSocket or SSE is on the roadmap and would significantly improve the user experience for customers waiting on approval decisions.

2. **Cache invalidation is time-based only.** If a customer uploads a document and immediately navigates to the documents page, they might see stale data for up to 60 seconds. A cache-busting query parameter is used as a workaround on the front-end, but proper event-driven invalidation would be cleaner.

3. **CORS configuration is permissive in staging.** The staging environment allows all origins to simplify QA. Production is locked down to the portal domain. Ensure you are testing against the correct environment if you encounter CORS issues.
