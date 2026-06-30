# Architecture: feature-flag-service

## Component Diagram

```
+-------------------+    +-------------------+    +-------------------+
| credit-decision-  |    | disbursement-     |    | customer-portal-  |
| service            |    | service            |    | bff               |
+--------+----------+    +--------+----------+    +--------+----------+
         |                         |                         |
         +------------+------------+-------------------------+
                      |
             REST (flag evaluation)
                      |
                      v
+------------------------------------------------------------------------+
|                    feature-flag-service                                 |
|                                                                        |
|  +---------------------+    +------------------------+                 |
|  | FlagController      |--->| FlagEvaluator          |                 |
|  | (REST API)          |    +---+--------+-----------+                 |
|  +---------------------+        |        |                             |
|                                 v        v                             |
|        +-------------------+  +------------------------+               |
|        | Redis             |  | LaunchDarkly SDK       |               |
|        | feature_flags_    |  | (upstream sync)        |               |
|        | cache             |  +------------------------+               |
|        +-------------------+                                           |
|                                                                        |
|  +---------------------+                                               |
|  | OverrideController  |  Local override API for testing               |
|  +---------------------+                                               |
+------------------------------------------------------------------------+
```

## Data Flow

1. **Flag Sync** -- A background goroutine connects to LaunchDarkly's streaming API and receives flag configuration updates in real time. Flag definitions and targeting rules are written to Redis (`feature_flags_cache`) with a TTL of 60 seconds as a safety net.
2. **Flag Evaluation** -- When a service calls `GET /api/v1/flags/{key}/evaluate` with a context payload (user ID, segment, environment), the `FlagEvaluator` first checks Redis for the cached flag state. If the cache is warm, evaluation happens locally in-process without contacting LaunchDarkly. Cache misses fall through to the LaunchDarkly SDK.
3. **Bulk Evaluation** -- Services can request `POST /api/v1/flags/evaluate-bulk` with a list of flag keys to reduce round trips. The response includes all requested flag values in a single payload.
4. **Local Overrides** -- In non-production environments, `OverrideController` allows engineers to set flag overrides via `PUT /api/v1/flags/{key}/override`. Overrides are stored in Redis with a separate key prefix and take precedence over LaunchDarkly values. Overrides are auto-expired after 24 hours.

## Integrations

| System | Direction | Protocol | Purpose |
|---|---|---|---|
| All CredityFlow services | Inbound | REST API | Feature flag evaluation |
| LaunchDarkly | Outbound | Streaming API | Flag configuration source |
| Redis (feature_flags_cache) | Internal | Redis protocol | Local flag cache and overrides |

## Architecture Decision Records

### ADR-052-1: Go Instead of Kotlin

**Status:** Accepted
**Context:** The feature-flag-service handles a high volume of evaluation requests (every service in the platform queries flags on most requests). Initial load testing with a Kotlin/Spring Boot prototype showed ~150ms cold-start times and ~80MB baseline memory per pod. The team needed a service with minimal latency overhead and low resource footprint, since flag evaluation is on the critical path of nearly every API call.
**Decision:** Implement the feature-flag-service in Go using the Gin framework. Go's compiled binary, fast startup (~10ms), low memory footprint (~15MB), and excellent concurrency model (goroutines for LaunchDarkly streaming) make it well-suited for this high-throughput, low-latency workload.
**Consequences:** This is the only Go service in the CredityFlow ecosystem, which means the team-platform engineers need to maintain Go expertise alongside Kotlin. To mitigate this, the service is intentionally kept small and simple (under 2,000 lines of code). The CI/CD pipeline required a separate Go build stage. The trade-off is justified by a 10x reduction in p99 latency (8ms vs 80ms) and 5x reduction in memory usage.

### ADR-052-2: Proxy LaunchDarkly Instead of Direct SDK

**Status:** Accepted
**Context:** LaunchDarkly provides client SDKs for direct integration. However, with 50+ services, each would need its own SDK instance, resulting in 50+ concurrent connections to LaunchDarkly, duplicated flag data in memory across every pod, and per-seat licensing costs that scale with service count.
**Decision:** Centralize flag evaluation behind the feature-flag-service, which maintains a single connection to LaunchDarkly and caches flag state in Redis. All other services call this proxy via REST instead of embedding the LaunchDarkly SDK.
**Consequences:** Single point of control for flag evaluation: audit logging, override capabilities, and consistent caching are centralized. LaunchDarkly costs are fixed regardless of service count. The trade-off is an additional network hop (~2-5ms) for flag evaluation, but this is mitigated by the Redis cache. If the feature-flag-service is unavailable, services fall back to a hardcoded default value per flag key, which is acceptable for short outages.
