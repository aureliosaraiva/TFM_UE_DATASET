# Backoffice BFF -- Architecture Document

**Service:** svc-46 | **Domain:** Backoffice | **Stack:** Node.js / Express | **Cache:** Redis

---

## What Is This Service?

The Backoffice BFF serves the internal operations dashboard used by loan officers, collections agents, compliance analysts, and platform administrators. Like its sibling (customer-portal-bff), it aggregates backend service responses into UI-friendly payloads. Unlike the customer BFF, it also manages review queues, supports bulk operations, and enforces role-based access at the API level.

## Key Differences from Customer Portal BFF

| Concern | Customer Portal BFF | Backoffice BFF |
|---------|-------------------|----------------|
| Auth model | Customer JWT (single role) | Internal JWT with RBAC (multiple roles/permissions) |
| Data scope | Single customer's data | Cross-customer, cross-contract queries |
| Write operations | Limited (accept offer, update profile) | Extensive (approve cases, override rules, trigger actions) |
| Review queues | None | Central feature |
| Concurrency | High (thousands of customers) | Lower (tens of internal users) but heavier queries |

## Components

### Review Queue Manager

The backoffice's most distinctive feature. The BFF maintains several review queues backed by Redis sorted sets:

- **Appraisal Review Queue** -- appraisals awaiting manual verification (flagged by business rules).
- **Renegotiation Approval Queue** -- renegotiation offers exceeding auto-approval thresholds.
- **Compliance Alert Queue** -- cases flagged by the audit service.
- **Legal Escalation Queue** -- legal cases needing agent action.

Each queue item includes a summary payload (pre-fetched and cached) so the agent sees context without additional API calls. Queue assignment supports round-robin and skill-based routing.

### RBAC Middleware

Every route is protected by a permission check against the JWT claims. Permissions are hierarchical:

```
admin > compliance_analyst > collections_manager > collections_agent > viewer
```

The middleware calls auth-service once per session to fetch the user's permission set and caches it in Redis for 10 minutes.

### Route Groups

- `/api/v1/cases` -- case management: list, filter, assign, update status.
- `/api/v1/queues` -- review queue operations: claim, release, complete.
- `/api/v1/contracts` -- contract detail with full history (installments, payments, appraisals, legal cases).
- `/api/v1/customers` -- customer 360 view: all contracts, all interactions, risk profile.
- `/api/v1/reports` -- on-demand operational reports (portfolio aging, collections performance).
- `/api/v1/admin` -- rule management proxies (eligibility rules, negotiation rules, dunning configs).

### Service Clients

Communicates with the same backend services as the customer BFF, plus additional backoffice-specific endpoints:
- case-management-service (core dependency)
- billing-service, dunning-service, legal-collections-service
- auth-service (user management, permissions)
- audit-service (compliance log queries)
- feature-flag-service

## Data Flow

```
Backoffice SPA (React)
      |
      | HTTPS (internal ALB, VPN-only)
      v
Backoffice BFF
      |
      +---> auth-service (RBAC validation)
      |
      +---> case-management-service (case CRUD)
      |
      +---> billing-service (contract data)
      |
      +---> dunning-service (campaign status)
      |
      +---> legal-collections-service (legal case data)
      |
      +---> audit-service (audit log queries)
      |
      +---> [other services as needed]
      |
      v
   JSON Response
```

The backoffice BFF is accessible only through the internal ALB, which requires VPN access. It is not exposed to the public internet.

## Integration Patterns

- **Synchronous aggregation:** Same fan-out pattern as the customer BFF, but with broader service coverage.
- **Write-through:** Write operations (case updates, rule changes, manual overrides) are forwarded to the appropriate backend service. The BFF adds audit metadata (who, when, from which IP) before forwarding.
- **Redis queues:** Review queues use Redis sorted sets (scored by priority) with atomic claim/release operations. Redis pub/sub notifies connected agents when new items enter their queue.
- **WebSocket (planned):** Real-time queue updates are currently achieved via polling (10s interval). Migration to WebSocket is on the roadmap.

## ADR

### ADR-046-001: BFF-managed review queues over a dedicated queue service

**Context:** Review queues could live in a dedicated microservice or be managed by the BFF. A dedicated service would be more aligned with microservice principles but adds operational overhead for a feature used only by the backoffice.

**Decision:** Manage review queues in the BFF using Redis. The queue logic is simple (sorted set operations) and tightly coupled to the backoffice UI's needs.

**Consequences:** Faster iteration on queue features (no cross-team coordination). Queue state is in Redis, not a durable database -- acceptable because queue items are derived from backend service state and can be rebuilt. If the BFF is restarted, queues are repopulated from backend services within 60 seconds.

### ADR-046-002: VPN-only access instead of public endpoint with strong auth

**Context:** The backoffice handles sensitive customer data and administrative operations. Access control options ranged from strong auth alone to network-level restrictions.

**Decision:** Restrict the backoffice BFF to VPN-only access at the network level (ALB security group), in addition to JWT-based RBAC. Defense in depth.

**Consequences:** Internal users must be on VPN to access the backoffice. Adds friction for remote work but significantly reduces attack surface. The customer portal BFF remains publicly accessible since customers cannot use VPN.
