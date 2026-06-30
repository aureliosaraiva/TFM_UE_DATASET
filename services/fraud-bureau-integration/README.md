# Fraud Bureau Integration

> **Service name:** `fraud-bureau-integration` — owned by `team-fraud`, part of the Fraud Prevention domain.

## Overview

The `fraud-bureau-integration` acts as CredityFlow's bridge to ClearCheck, an external fraud bureau that provides fraud intelligence data. It is a stateless integration layer -- it does not maintain its own database. Its job is simple in concept but tricky in practice: take a customer's identity, ask ClearCheck if there are any fraud signals, and normalize the response into something the rest of our fraud pipeline can understand.

The service is owned by team-fraud and is part of the Fraud Prevention domain.

## Getting Started

### Prerequisites

- JDK 17+
- A ClearCheck sandbox API key (ask team-fraud for access to the shared sandbox credentials)
- Running instance of customer-service (or the mock -- see below)

### Running Locally

```bash
# Set ClearCheck sandbox credentials
export CLEARCHECK_API_KEY=sandbox_key_here
export CLEARCHECK_API_URL=https://sandbox.clearcheck.com.br/v3

# Start the service
./gradlew bootRun --args='--spring.profiles.active=local'
```

The service runs on port `8095`. There's no Swagger UI configured since this is an internal-only service, but you can hit the endpoint directly:

```bash
curl -X POST http://localhost:8095/api/v1/fraud-bureau/check \
  -H "Content-Type: application/json" \
  -d '{"customerId": "uuid-here", "proposalId": "uuid-here"}'
```

### Using the ClearCheck Mock

For development without ClearCheck sandbox access, enable the mock profile:

```bash
./gradlew bootRun --args='--spring.profiles.active=local,mock-clearcheck'
```

The mock returns configurable responses. See `src/test/resources/clearcheck-mock-responses/` for the available scenarios.

### Tests

```bash
./gradlew test
```

The integration tests against ClearCheck are tagged with `@Tag("clearcheck-sandbox")` and are excluded from the default test suite. To run them:

```bash
./gradlew test -PincludeClearCheck=true
```

You will need valid sandbox credentials.

## Architecture

The service is intentionally stateless. It receives a request, calls customer-service to get identity data, translates that into ClearCheck's format, calls ClearCheck, translates the response back, and publishes an event. No data is persisted locally.

### Key Components

- **FraudBureauController** - REST endpoint for internal callers
- **FraudBureauEventListener** - Kafka consumer for `biometrics_validated` events
- **ClearCheckClient** - HTTP client with circuit breaker, retry logic, and timeout handling
- **ClearCheckMapper** - Bidirectional mapper between CredityFlow and ClearCheck data models
- **FraudSignalNormalizer** - Converts ClearCheck-specific signal types into CredityFlow's standardized `FraudSignal` taxonomy

### Circuit Breaker

The ClearCheck client uses Resilience4j with the following configuration:
- **Failure threshold:** 3 consecutive failures opens the circuit
- **Wait duration:** 60 seconds in open state before attempting half-open
- **Permitted calls in half-open:** 2

When the circuit is open, the service publishes a `fraud_bureau_checked` event with status `DEGRADED` and an empty signals list. The fraud-scoring-service handles this by computing a score without bureau signals (which tends to be more conservative).

### Dealing with ClearCheck Quirks

A few things worth knowing about ClearCheck's API:

1. **Rate limiting:** ClearCheck enforces 100 req/s. We have a client-side rate limiter set to 80 req/s to provide headroom. If you see 429 responses in logs, the rate limiter configuration may need adjusting.

2. **Response time variability:** ClearCheck's response time varies between 200ms and 3s depending on load. Our timeout is set to 8s. Do not lower this -- they have confirmed that some complex queries can take up to 6s.

3. **Schema versioning:** ClearCheck uses a `X-API-Version` header. We currently pin to `v3`. They deprecated `v2` in Q3 2024 with 30 days notice. When they announce v4, we will need to update the mapper.

4. **Test CPFs:** ClearCheck's sandbox recognizes specific test CPFs that return known signal patterns. These are documented in `src/test/resources/clearcheck-test-cpfs.json`.

## API Reference

### POST /api/v1/fraud-bureau/check

**Request:**
```json
{
  "customerId": "550e8400-e29b-41d4-a716-446655440000",
  "proposalId": "660e8400-e29b-41d4-a716-446655440001"
}
```

**Response (200):**
```json
{
  "checkId": "uuid",
  "status": "COMPLETED",
  "signalCount": 2,
  "signals": [
    {
      "type": "VELOCITY_ALERT",
      "severity": "MEDIUM",
      "description": "Multiple credit applications in short timeframe",
      "source": "CLEARCHECK"
    },
    {
      "type": "ADDRESS_INCONSISTENCY",
      "severity": "LOW",
      "description": "Declared address does not match bureau records",
      "source": "CLEARCHECK"
    }
  ],
  "checkedAt": "2025-01-15T10:30:00Z"
}
```

**Response when ClearCheck unavailable (200 with degraded status):**
```json
{
  "checkId": "uuid",
  "status": "DEGRADED",
  "signalCount": 0,
  "signals": [],
  "checkedAt": "2025-01-15T10:30:00Z"
}
```

Note: We return 200 even in degraded mode because from the caller's perspective the check completed -- it just completed without external data. This is a deliberate design decision (see ADR-001 in architecture.md).

## Configuration

| Property | Default | Description |
|---|---|---|
| `clearcheck.api-url` | - | ClearCheck API base URL |
| `clearcheck.api-key` | - | ClearCheck API key (from vault) |
| `clearcheck.timeout-ms` | `8000` | HTTP request timeout |
| `clearcheck.rate-limit-per-second` | `80` | Client-side rate limit |
| `clearcheck.circuit-breaker.failure-threshold` | `3` | Failures before circuit opens |
| `clearcheck.circuit-breaker.wait-duration-s` | `60` | Wait time in open state |

## Deployment

Deployed in the `fraud` namespace. Since this service is stateless, it scales horizontally with no concerns about data consistency. Current production runs 3 replicas.

Resource requirements are modest -- 256MB memory, 0.25 CPU per pod.

The service needs outbound network access to ClearCheck's API (whitelisted IPs are configured at the infrastructure level).

## Monitoring

- **Fraud Bureau Integration Health** dashboard shows ClearCheck call success rate, latency, and circuit breaker state
- Key alert: circuit breaker open for > 5 minutes triggers a P2 incident

## Known Issues

- **ClearCheck sandbox instability:** The sandbox environment goes down occasionally for maintenance without advance notice. This does not affect production.
- **Signal taxonomy drift:** ClearCheck occasionally adds new signal types that our normalizer does not recognize. Unknown signals are mapped to `UNKNOWN` type and logged with a warning. When you see these warnings, add a mapping to `FraudSignalNormalizer`.
- **No retry on event publish failure:** If the Kafka publish of `fraud_bureau_checked` fails after a successful ClearCheck call, the check result is lost. The fraud pipeline will eventually timeout and retry the entire flow. This is a known gap that has not been prioritized since the failure rate is very low (<0.01%).
