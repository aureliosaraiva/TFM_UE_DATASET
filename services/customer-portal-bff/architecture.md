# Customer Portal BFF

**svc-45 | Customer Portal Domain | Node.js / Express | Redis**

## Role

The Customer Portal BFF (Backend for Frontend) is the single entry point for the CredityFlow customer-facing web and mobile applications. It aggregates data from multiple backend microservices, shapes responses for frontend consumption, and handles session management.

It contains no business logic. Its job is to compose, transform, and cache responses so the frontend makes a single call instead of orchestrating multiple backend requests.

## Architecture

### Why a BFF?

CredityFlow's backend is decomposed into fine-grained microservices. Without a BFF, the customer portal frontend would need to:
- Call 3-5 services to render a single dashboard page.
- Handle partial failures across those calls.
- Manage authentication tokens for each service.
- Deal with response formats optimized for service-to-service communication, not UI rendering.

The BFF solves all of these by providing a UI-optimized API layer.

### Stack Choice: Node.js

The BFF is one of two Node.js services in the CredityFlow ecosystem (alongside backoffice-bff). Node.js was chosen for its strengths in I/O-bound aggregation workloads -- the BFF spends most of its time waiting for backend responses, and Node's event loop handles concurrent outbound calls efficiently without thread-pool tuning.

### Components

**Route Handlers** -- Express routes organized by customer-facing feature:
- `/api/v1/dashboard` -- aggregates contract summary, upcoming payments, and notifications.
- `/api/v1/contracts/:id` -- contract detail with installment schedule and appraisal status.
- `/api/v1/renegotiation` -- eligibility check and simulation orchestration.
- `/api/v1/documents` -- lists and provides download URLs for reports and boletos.

**Service Clients** -- Thin HTTP clients for each backend service, with Axios interceptors for:
- JWT forwarding (passes the customer's auth token to backend services).
- Circuit breaking (opossum library, per-service configuration).
- Request/response logging (sanitized, no PII in logs).

**Response Composers** -- Transform and merge responses from multiple services into frontend-optimized shapes. Handle partial failures by returning degraded responses (e.g., dashboard without notifications if notification-service is down).

**Redis Cache Layer** -- Caches aggregated responses with short TTLs:
- Dashboard: 60 seconds.
- Contract schedule: 5 minutes (changes infrequently).
- Eligibility results: not cached (must be real-time).

**Session Middleware** -- Validates JWT tokens issued by auth-service. Extracts customer ID and permissions. Does not issue tokens -- that is auth-service's responsibility.

## Data Flow

```
Mobile/Web App
      |
      | HTTPS (API Gateway / ALB)
      v
Customer Portal BFF (Node.js)
      |
      +---> auth-service (JWT validation, per-request)
      |
      +---> billing-service (schedule, balance)
      |
      +---> notification-service (unread notifications)
      |
      +---> appraisal-report-service (document URLs)
      |
      +---> renegotiation-eligibility-service (eligibility)
      |
      +---> renegotiation-simulation-service (simulations)
      |
      +---> feature-flag-service (feature toggles)
      |
      v
   JSON Response (UI-optimized)
```

All backend calls are made in parallel where possible (Promise.all). The BFF waits up to 5 seconds for all responses; any service that exceeds the timeout is excluded from the response with a degraded flag.

## Integration Patterns

- **Synchronous fan-out:** The BFF makes parallel REST calls to backend services and merges results. This is the defining pattern of the service.
- **No event production or consumption:** The BFF is purely synchronous. It does not subscribe to any SNS/SQS topics.
- **Feature flags:** Checks feature-flag-service at request time to determine which UI features to expose. Flags are cached in Redis for 30 seconds to avoid per-request calls.
- **Rate limiting:** Applied at the API Gateway level (AWS ALB + WAF), not in the BFF itself.
- **CORS:** Configured to allow only the customer portal domain and mobile app user-agent.

## ADR

### ADR-045-001: Dedicated BFF per frontend over a shared API gateway

**Context:** A shared GraphQL gateway was considered as an alternative to separate BFFs for customer portal and backoffice. GraphQL would let each frontend query exactly what it needs.

**Decision:** Use dedicated BFFs. The customer portal and backoffice have fundamentally different security models (customer auth vs. internal RBAC), different data access patterns, and different SLA requirements. A shared gateway would become a coupling point.

**Consequences:** Two BFF services to maintain instead of one gateway. Each can evolve independently and have tailored caching, error handling, and security policies. The tradeoff is some duplication of backend service client code, mitigated by a shared npm package (`@credityflow/service-clients`).

### ADR-045-002: Graceful degradation over hard failure

**Context:** The dashboard page calls 4 backend services. If one is down, should the BFF return a 500 or a partial response?

**Decision:** Return partial responses with a `degraded` flag indicating which sections are unavailable. The frontend renders available sections and shows a retry prompt for degraded ones.

**Consequences:** Improved perceived availability -- the customer sees their balance even if notifications are down. Increases frontend complexity (must handle partial responses). Requires clear contracts between BFF and frontend on which fields are optional.
