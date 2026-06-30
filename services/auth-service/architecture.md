# Architecture: auth-service

## Component Diagram

```
                     +---------------------------+
                     | customer-portal-bff       |
                     | backoffice-bff            |
                     +-------------+-------------+
                                   |
                          REST (login, refresh, validate)
                                   |
                                   v
+------------------------------------------------------------------------+
|                         auth-service                                   |
|                                                                        |
|  +---------------------+    +------------------------+                 |
|  | AuthController      |--->| AuthenticationService  |                 |
|  | (REST API)          |    +---+--------+-----------+                 |
|  +---------------------+        |        |                             |
|                                 v        v                             |
|        +-------------------+  +------------------+                     |
|        | TokenService      |  | RBACService      |                     |
|        | (JWT issue/verify)|  | (roles/perms)    |                     |
|        +--------+----------+  +--------+---------+                     |
|                 |                       |                               |
|                 v                       v                               |
|        +-------------------+  +------------------+                     |
|        | Redis             |  | PostgreSQL       |                     |
|        | auth_session_cache|  | auth_db          |                     |
|        +-------------------+  +------------------+                     |
+------------------------------------------------------------------------+
```

## Data Flow

1. **Login Request** -- A user (customer or internal operator) submits credentials via the customer-portal-bff or backoffice-bff. The `AuthController` delegates to `AuthenticationService`.
2. **Credential Validation** -- `AuthenticationService` loads the `User` record from PostgreSQL (`auth_db`), verifies the hashed password, and checks account status (active, locked, etc.). Brute-force protection increments a failed-attempt counter and locks the account after 5 consecutive failures.
3. **Token Issuance** -- On successful authentication, `TokenService` generates a JWT access token (15-minute TTL) and a refresh token (7-day TTL). The refresh token is stored in Redis (`auth_session_cache`) with the session metadata.
4. **Token Refresh** -- When the access token expires, the client presents the refresh token. `TokenService` validates it against Redis, issues a new access token, and extends the refresh token TTL using a sliding window strategy. This keeps active users logged in without re-authentication.
5. **Token Validation** -- Downstream services call `GET /api/v1/auth/validate` with the Bearer token. `TokenService` verifies the JWT signature and checks Redis for session revocation. `RBACService` resolves the user's roles and permissions from PostgreSQL, returning them in the validation response.

## Integrations

| System | Direction | Protocol | Purpose |
|---|---|---|---|
| customer-portal-bff | Inbound | REST API | Customer login and token operations |
| backoffice-bff | Inbound | REST API | Internal user login and token operations |
| All services | Inbound | REST API | Token validation requests |
| PostgreSQL (auth_db) | Internal | JDBC | Users, roles, permissions, audit log |
| Redis (auth_session_cache) | Internal | Redis protocol | Session cache, refresh tokens, lockout counters |

## Architecture Decision Records

### ADR-051-1: JWT Over Session-Based Authentication

**Status:** Accepted
**Context:** CredityFlow runs 50+ microservices on AWS EKS. A traditional session-based authentication approach would require every service to call back to a centralized session store on every request, creating a single point of failure and a latency bottleneck. Sticky sessions were considered but conflict with Kubernetes horizontal pod autoscaling and rolling deployments.
**Decision:** Use JWT (JSON Web Tokens) for authentication. The auth-service issues signed JWTs containing the user identity and role claims. Downstream services verify the token signature locally using a shared public key, without calling auth-service on every request. Refresh tokens are stored in Redis for revocation support.
**Consequences:** Services can validate tokens independently, enabling stateless horizontal scaling. The trade-off is that a revoked token remains valid until its short TTL expires (max 15 minutes). For sensitive operations (e.g., disbursement approval), services additionally check Redis for real-time revocation status. JWT payload size is larger than a session cookie but acceptable for service-to-service calls.

### ADR-051-2: Redis for Session Cache

**Status:** Accepted
**Context:** Refresh token storage and session metadata require sub-millisecond reads for token validation and the ability to instantly invalidate sessions (e.g., password change, suspicious activity). PostgreSQL can handle this but adds latency and connection pressure at the request-per-second volume seen across the platform.
**Decision:** Use Redis (`auth_session_cache`) as the primary store for active sessions, refresh tokens, and brute-force lockout counters. Redis TTL is used for automatic session expiry. PostgreSQL remains the system of record for user accounts and RBAC configuration.
**Consequences:** Session validation is fast (~1ms). Redis failure degrades gracefully: JWT validation still works (signature-based), but refresh and revocation are temporarily unavailable. Redis is deployed as an ElastiCache cluster with automatic failover. Memory usage is bounded by session count and TTL configuration.
