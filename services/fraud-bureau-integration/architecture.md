# Fraud Bureau Integration (svc-15)

**Domain:** Fraud
**Stack:** Kotlin / Spring Boot
**Database:** None (stateless)
**Deployment:** AWS EKS

## Component Overview

The Fraud Bureau Integration service is a thin, stateless adapter that connects CredityFlow to ClearCheck, an external fraud bureau API widely used in Brazilian financial services. ClearCheck provides data on known fraudulent CPFs, addresses, phone numbers, and email addresses, as well as a composite fraud risk score based on their proprietary database.

This service is deliberately kept minimal. It translates internal domain requests into ClearCheck's API format, handles authentication and rate limiting, manages circuit breaking for the external dependency, and publishes the results as events. It stores nothing.

### Internals

**Request Transformer:** Converts an internal `FraudBureauCheckRequested` event payload (which uses CredityFlow's domain language and data structures) into ClearCheck's API request format. This includes mapping field names, formatting CPF with/without mask, and assembling the required authentication headers (HMAC-SHA256 signed requests).

**ClearCheck Client:** A resilient HTTP client built with Spring WebClient and enhanced with Resilience4j for:
- Circuit breaker (opens after 5 consecutive failures, half-open after 30 seconds).
- Retry with exponential backoff (max 3 attempts).
- Timeout (8 seconds per request).
- Bulkhead (max 20 concurrent requests to respect ClearCheck's rate limits).

**Response Normalizer:** Maps ClearCheck's response into CredityFlow's internal fraud signal format. ClearCheck returns scores on a 0-1000 scale; this is normalized to 0-100 for consistency with other fraud signals.

**Dead Letter Handling:** If all retries are exhausted, the original SQS message is sent to a dead letter queue (DLQ). An alarm on the DLQ triggers an alert to the operations team.

## Data Flow

1. The Fraud Decision Service publishes a `FraudBureauCheckRequested` event to SNS when a proposal reaches the fraud analysis stage.
2. This service picks up the message from its SQS subscription.
3. The request is transformed and sent to the ClearCheck API over HTTPS.
4. The response is normalized and published as a `FraudBureauCheckCompleted` event to SNS, containing the bureau's fraud indicators and normalized score.
5. The Fraud Scoring Service consumes this event as one of its three input signals.

The entire flow is stateless. If the service restarts mid-processing, the SQS message becomes visible again after the visibility timeout and is reprocessed. Idempotency is guaranteed because ClearCheck's API is itself idempotent for the same CPF within a 24-hour window.

## Integration Patterns

- **Inbound:** SQS queue subscribed to `FraudBureauCheckRequested` events.
- **Outbound (async):** Publishes `FraudBureauCheckCompleted` events to SNS.
- **External (sync):** HTTPS calls to ClearCheck's REST API. Mutual TLS authentication. IP whitelisting on ClearCheck's side restricts access to CredityFlow's NAT Gateway IPs.

## ADR-001: Stateless Adapter vs. Caching Bureau Results

**Context:** ClearCheck charges per API call. The team considered caching bureau results in Redis or PostgreSQL to avoid duplicate lookups for the same CPF. However, ClearCheck's data changes frequently (they receive real-time fraud reports), so cached results could be stale.

**Decision:** Keep the service stateless with no caching. Every fraud check results in a fresh ClearCheck API call. The cost (~R$0.15 per call) is negligible compared to the risk of using stale fraud data for a lending decision.

**Consequences:**
- Always-fresh fraud data from the bureau.
- Higher API costs, but acceptable at current volumes (~3,000 checks/day).
- Simpler architecture: no cache invalidation logic, no stale data bugs, no additional infrastructure.

## ADR-002: Circuit Breaker Configuration for External Dependency

**Context:** ClearCheck has a published SLA of 99.5% uptime, which means up to ~3.6 hours of downtime per month. During ClearCheck outages, CredityFlow's fraud analysis pipeline would stall entirely without protective measures.

**Decision:** Implement Resilience4j circuit breaker with the following configuration: open after 5 consecutive failures, stay open for 30 seconds, then half-open (allow 1 request through). When the circuit is open, the service publishes a `FraudBureauCheckUnavailable` event, allowing the Fraud Decision Service to proceed with the other two fraud signals (device fingerprint and internal scoring) and flag the proposal for deferred bureau check.

**Consequences:**
- CredityFlow can continue processing proposals during ClearCheck outages, with reduced fraud signal coverage.
- Proposals processed without bureau data are flagged for a retroactive check once ClearCheck recovers (handled by a scheduled reconciliation job).
- The 30-second open window prevents request pileup during extended outages.
