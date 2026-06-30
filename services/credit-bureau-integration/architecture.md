# Credit Bureau Integration (svc-18)

**Domain:** Credit
**Stack:** Kotlin / Spring Boot
**Database:** Redis (cache, TTL 24h)
**Deployment:** AWS EKS

## Component Overview

This service bridges CredityFlow with ServicosBureau, a major Brazilian credit bureau that provides credit history, outstanding debts, credit score, and payment behavior data for individuals and businesses. It follows the adapter pattern: translating between CredityFlow's internal event-driven architecture and ServicosBureau's synchronous SOAP/XML API.

Yes, SOAP. ServicosBureau still runs on SOAP in 2026. The service wraps this complexity so that no other part of CredityFlow needs to know about XML schemas, WSDL definitions, or WS-Security headers.

### Architecture

**SOAP Client Layer:** Built with Spring's `WebServiceTemplate`. Handles WSDL-based request/response serialization, WS-Security (UsernameToken + Timestamp), mutual TLS certificate management, and SOAP fault parsing. The WSDL is cached locally to avoid runtime dependency on ServicosBureau's WSDL endpoint.

**Response Transformer:** Converts ServicosBureau's verbose XML response (often 50+ fields, nested structures, legacy encodings) into a clean internal `CreditBureauReport` domain object. Null handling is particularly thorny -- ServicosBureau uses a mix of absent elements, empty strings, and the literal text "N/A" to represent missing data.

**Redis Cache:** Bureau queries for the same CPF within a 24-hour window return the cached result. This serves two purposes: cost reduction (ServicosBureau charges per query, and the price is non-trivial) and latency improvement (cache hits return in <5ms vs. 2-3 seconds for a live bureau call). The 24-hour TTL was negotiated with the credit risk team as an acceptable staleness window.

**Resilience Layer:** Circuit breaker (Resilience4j), retry with backoff, and timeout management. ServicosBureau's API is notoriously slow under load, with P99 latencies reaching 8 seconds. The circuit breaker opens after 3 consecutive timeouts.

## Data Flow

1. `CreditBureauCheckRequested` event arrives on the SQS queue (published by the Credit Decision Service when a proposal enters credit analysis).
2. Service checks Redis for a cached report for this CPF.
3. On cache miss: constructs the SOAP request, calls ServicosBureau, parses the XML response, transforms it into the internal format, and caches the result in Redis with 24h TTL.
4. On cache hit: retrieves the cached report directly.
5. Publishes `CreditBureauReportReady` event to SNS with the full report payload.

## Integration Patterns

- **Inbound (async):** SQS subscription to `CreditBureauCheckRequested` events.
- **Outbound (async):** `CreditBureauReportReady` events to SNS, consumed by `income-verification-service`, `credit-scoring-engine`, and `credit-policy-service`.
- **External (sync):** SOAP/HTTPS to ServicosBureau. Mutual TLS. IP whitelisting. Production credentials rotated quarterly.
- **Redis:** Cache layer for cost and latency optimization.

## ADR-001: 24-Hour Cache TTL for Bureau Data

**Context:** ServicosBureau charges R$2.50 per query. During the early months, duplicate queries for the same CPF (caused by proposal retries, re-submissions, and multiple services requesting the same data) were inflating costs. The credit team confirmed that bureau data changes infrequently enough that a 24-hour staleness window is acceptable for lending decisions.

**Decision:** Cache bureau responses in Redis with a 24-hour TTL. The cache key is the CPF hash. Any service requesting bureau data within 24 hours of the first query gets the cached result.

**Consequences:**
- ~35% reduction in bureau API costs in the first month after implementation.
- Rare edge case: a customer's credit situation could change between the cached query and the lending decision. Accepted risk, since the final credit decision also incorporates internal scoring and policy rules.
- Cache invalidation is manual: if the credit team learns that a specific CPF's data is stale, they can force a cache eviction through the admin API.

## ADR-002: Encapsulating SOAP Complexity in a Dedicated Service

**Context:** The initial approach had the Credit Scoring Engine calling ServicosBureau directly. This leaked SOAP/XML handling into a service focused on ML scoring, creating maintenance burden and making testing difficult (mocking SOAP responses is painful).

**Decision:** Extract all ServicosBureau communication into a dedicated integration service. This service is the sole point of contact with the bureau. All other services consume the clean internal event.

**Consequences:**
- ServicosBureau's API quirks (SOAP, XML, legacy encodings) are isolated in one place.
- If ServicosBureau ever upgrades to REST (they have been promising this since 2023), only this service needs to change.
- The credit domain services can be tested with simple event fixtures instead of SOAP mocks.
