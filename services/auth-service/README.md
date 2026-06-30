# Auth Service

> **svc-51** -- Platform domain, owned by **team-platform**

## Overview

Auth Service is the identity and authentication gateway for the entire CredityFlow ecosystem. It issues and validates **JSON Web Tokens (JWT)** for two distinct user populations:

- **Customers** accessing the self-service portal
- **Internal users** (back-office agents, compliance officers, administrators)

The service manages credential storage, session lifecycle, token refresh, and logout/revocation. It does *not* handle authorization policies -- those are enforced at the service mesh and individual service level using claims embedded in the JWT.

**Tech stack:** Kotlin, Spring Boot, Spring Security
**Primary store:** PostgreSQL (`auth_db`) for credentials and user records
**Session cache:** Redis (`auth_session_cache`) for active session tracking and token blacklisting

> This service stores **PII** including hashed credentials, email addresses, and phone numbers. It is classified as a Tier-0 critical service.

## Getting Started

Prerequisites: JDK 17+, Docker Compose, Gradle wrapper.

```bash
# bring up Postgres and Redis
docker compose up -d

# run locally
./gradlew bootRun -Dspring.profiles.active=local

# run tests
./gradlew test
```

The service starts on port **8051** by default. A seeded admin user is created in the `local` profile -- see `src/main/resources/db/seed/local.sql` for credentials.

## Architecture

```
                        +------------------+
     Login / Refresh    |                  |    Validate
    ─────────────────>  |   Auth Service   |  <─────────────
    (customer / agent)  |                  |  (other services)
                        +--------+---------+
                                 |
                  +--------------+--------------+
                  |                             |
          +-------v--------+          +--------v--------+
          |  PostgreSQL     |          |  Redis           |
          |  auth_db        |          |  auth_session    |
          |  - users        |          |  _cache          |
          |  - credentials  |          |  - active tokens |
          |  - login_audit  |          |  - blacklist     |
          +----------------+          +-----------------+
```

**Token lifecycle:**

1. Client sends credentials to `/login`.
2. Service validates against `auth_db`, issues a JWT access token (short-lived, 15 min) and a refresh token (longer-lived, stored in Redis).
3. Other services call `/validate` on every request to verify the JWT signature and check the Redis blacklist.
4. Client calls `/refresh` before the access token expires.
5. On `/logout`, both tokens are blacklisted in Redis.

## API Reference

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/auth/login` | Authenticate with email + password. Returns JWT access and refresh tokens. |
| `POST` | `/api/v1/auth/refresh` | Exchange a valid refresh token for a new access token. |
| `GET` | `/api/v1/auth/validate` | Validate an access token (called service-to-service). Returns decoded claims or 401. |
| `POST` | `/api/v1/auth/logout` | Invalidate the current session. Blacklists both tokens. |

Error codes follow CredityFlow's standard `{ "error": "CODE", "message": "..." }` envelope.

## Configuration

| Variable | Purpose | Default |
|----------|---------|---------|
| `AUTH_DB_URL` | JDBC connection string for `auth_db` | `jdbc:postgresql://localhost:5432/auth_db` |
| `AUTH_DB_USERNAME` / `AUTH_DB_PASSWORD` | Database credentials | `auth` / `auth` |
| `REDIS_HOST` / `REDIS_PORT` | Redis for session cache | `localhost` / `6379` |
| `JWT_SECRET` | HMAC secret for signing tokens (prod uses RSA keypair via Vault) | -- |
| `JWT_ACCESS_TTL_SECONDS` | Access token time-to-live | `900` |
| `JWT_REFRESH_TTL_SECONDS` | Refresh token time-to-live | `86400` |
| `BCRYPT_STRENGTH` | bcrypt work factor | `12` |
| `LOGIN_RATE_LIMIT` | Max failed login attempts before temporary lock | `5` |

In production, secrets are injected from **AWS Secrets Manager** via the EKS CSI driver.

## Deployment

Deployed to **AWS EKS** in the `platform` namespace.

```bash
helm upgrade --install auth-service infra/helm/auth-service \
  --namespace platform \
  -f values-production.yaml
```

**Production sizing:**
- 3 replicas (anti-affinity across AZs)
- 256Mi-512Mi memory, 250m-500m CPU
- Horizontal Pod Autoscaler targets 70% CPU

**Zero-downtime deployments** are supported. Rolling update strategy ensures at least 2 replicas are available during deploys.

Database migrations run automatically on startup via Flyway. Backwards-compatible migrations only -- destructive changes require a two-phase deploy.

## Monitoring

- **Grafana dashboard:** Platform / Auth Service
- **Health endpoint:** `GET /actuator/health` (Postgres + Redis checks)
- **Key metrics:**
  - `auth.login.success_total` / `auth.login.failure_total` -- track login success rate
  - `auth.token.validate_latency_ms` -- p50/p95/p99 for validation calls (should be < 5ms p99)
  - `auth.session.active_count` -- gauge of active sessions in Redis
  - `auth.login.locked_accounts` -- counter of accounts temporarily locked due to brute-force protection

**Alerts:**
- Login failure rate > 20% over 5 minutes (possible credential stuffing)
- Validation latency p99 > 50ms (Redis connectivity issue)
- Redis connection pool exhaustion

## Known Issues

1. **Refresh token rotation not yet implemented.** Currently a refresh token can be reused until it expires. Rotation (invalidate-on-use) is planned to limit the blast radius of a leaked refresh token.

2. **No MFA support.** Multi-factor authentication is on the roadmap but not yet available. High-privilege internal accounts should use VPN + SSO as a compensating control until MFA ships.

3. **Session cache cold start.** If Redis is flushed, all active sessions are invalidated and users must re-authenticate. This is by design (fail-closed), but it has caused support ticket spikes during Redis maintenance windows. Consider scheduling Redis maintenance during low-traffic hours.
