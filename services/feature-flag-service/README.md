# Feature Flag Service

```
Service:    svc-52
Domain:     Platform
Team:       team-platform
Language:   Go (Gin framework)
Storage:    Redis (feature_flags_cache)
```

## Overview

The Feature Flag Service is the **only Go-based service** in the CredityFlow microservices ecosystem. It acts as a caching proxy in front of **LaunchDarkly**, providing a fast, low-latency local lookup for feature flags without requiring every service to integrate the LaunchDarkly SDK directly.

Why a proxy instead of direct SDK integration?

- **Consistency.** All services query flags through a single point, ensuring consistent evaluation across the fleet.
- **Cost.** LaunchDarkly pricing scales with connected SDK instances. Centralizing flag evaluation through one service drastically reduces the seat count.
- **Override capability.** The service supports local overrides, letting engineers force flags to specific values in non-production environments without touching LaunchDarkly.
- **Resilience.** Redis caching means flags remain available even if LaunchDarkly's CDN is unreachable. The last known flag state is served until connectivity is restored.

## Getting Started

You will need Go 1.21+ and a running Redis instance.

```bash
# install deps
go mod download

# start Redis
docker compose up -d redis

# run the service (LaunchDarkly SDK key optional for local dev)
LAUNCHDARKLY_SDK_KEY=sdk-local-dev \
REDIS_ADDR=localhost:6379 \
go run cmd/server/main.go

# alternatively
make run

# tests
make test
```

The service binds to port **8052**. Without a valid LaunchDarkly SDK key, it operates in "local mode" using only Redis-stored overrides.

## Architecture

```
  +-----------------+        +-----------------+
  | LaunchDarkly    | -----> | Feature Flag    | <---- Other CredityFlow services
  | (upstream)      |  sync  | Service (Go)    |       GET /api/v1/flags/{key}
  +-----------------+        +--------+--------+
                                      |
                                      v
                              +-------+-------+
                              | Redis         |
                              | flags_cache   |
                              | + overrides   |
                              +---------------+
```

**Sync loop:** A background goroutine polls LaunchDarkly every 30 seconds (configurable), fetches the full flag ruleset, and writes it to Redis. This decouples request-serving from external API availability.

**Evaluation:** When a service queries a flag, the handler reads from Redis first. If the flag has a local override, that value wins. Otherwise, the cached LaunchDarkly value is returned. If neither exists, a sensible default (provided by the caller) is used.

**Override layer:** Overrides are stored in a separate Redis hash (`flag_overrides`). They take precedence over LaunchDarkly values and are intended for testing and staged rollouts in lower environments. In production, overrides require an admin-scoped JWT.

## API Reference

### Evaluate a Single Flag

```
GET /api/v1/flags/{flag_key}?default=false&context=customer:abc-123
```

Returns:

```json
{
  "key": "enable_new_credit_model",
  "value": true,
  "source": "launchdarkly",
  "cached_at": "2026-03-29T10:15:00Z"
}
```

The `source` field indicates whether the value came from `launchdarkly`, `override`, or `default`.

### List All Flags

```
GET /api/v1/flags
```

Returns all known flags with their current values. Supports `?prefix=` to filter by naming convention (e.g., `?prefix=credit_` for all credit-domain flags).

### Set Override

```
PUT /api/v1/flags/{flag_key}/override
Content-Type: application/json

{ "value": true, "reason": "Testing new flow in staging" }
```

Sets a local override for the given flag. Requires `flags:admin` scope in the JWT. To remove an override, send `{ "value": null }`.

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `PORT` | HTTP listen port | `8052` |
| `REDIS_ADDR` | Redis address | `localhost:6379` |
| `REDIS_PASSWORD` | Redis password | (empty) |
| `REDIS_DB` | Redis database number | `0` |
| `LAUNCHDARKLY_SDK_KEY` | LaunchDarkly server-side SDK key | -- |
| `LD_SYNC_INTERVAL_SECONDS` | How often to sync from LaunchDarkly | `30` |
| `CACHE_TTL_SECONDS` | TTL for cached flag values | `300` |
| `OVERRIDE_ALLOWED_ENVS` | Comma-separated envs where overrides are permitted without admin scope | `local,staging` |
| `LOG_LEVEL` | Logging verbosity | `info` |

## Deployment

Deployed to AWS EKS in the `platform` namespace.

```bash
helm upgrade --install feature-flag-service infra/helm/feature-flag-service \
  --namespace platform -f values-production.yaml
```

This is a lightweight service:
- 2 replicas
- 64Mi-128Mi memory
- 50m-100m CPU
- Startup probe on `GET /healthz`

The Go binary compiles to a ~15MB static binary in a distroless container image. Cold start is under 2 seconds.

## Monitoring

Health check: `GET /healthz` (Redis connectivity + LaunchDarkly sync recency).

Prometheus metrics exposed at `/metrics`:

| Metric | Type | Description |
|--------|------|-------------|
| `flags_requests_total` | counter | Flag evaluations by key and source |
| `flags_cache_hits` | counter | Requests served from Redis cache |
| `flags_cache_misses` | counter | Cache misses (fallback to default) |
| `flags_ld_sync_duration_seconds` | histogram | Time taken for LaunchDarkly sync |
| `flags_ld_sync_errors_total` | counter | Failed sync attempts |
| `flags_overrides_active` | gauge | Number of active overrides |

Dashboard: **Grafana > Platform > Feature Flags**

Alerts:
- LaunchDarkly sync failures for > 5 minutes
- Cache miss rate > 20% (indicates flag naming mismatches)

## Known Issues

1. **No per-user flag targeting.** The service evaluates flags globally, not per-user. LaunchDarkly's user-targeting rules are flattened to a single boolean/string value during sync. If you need per-user segmentation, you'll need to pass a `context` parameter and the service will forward it to LaunchDarkly for evaluation -- but this path bypasses the cache and adds latency.

2. **Override cleanup is manual.** Overrides do not expire automatically. Stale overrides in staging have caused confusion when behaviour diverges from production. A TTL for overrides is a frequently requested improvement.

3. **Go dependency management.** Since this is the only Go service, the team has less familiarity with Go module management and tooling. CI pipelines for this service use a different base image and build process from the rest of the fleet.
