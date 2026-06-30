# Biometrics Capture Service

## Overview

The Biometrics Capture Service is the orchestrator of CredityFlow's identity verification pipeline. When a customer submits their documents as part of a loan application, this service kicks off the biometric verification chain: selfie capture, liveness detection, and face matching against the ID document photo.

It does not perform liveness or face matching itself -- those are handled by dedicated downstream services (liveness-worker and face-match-service). Instead, it owns the `BiometricSession` state machine and coordinates the flow by publishing and consuming events at each stage. Think of it as the conductor; the musicians are elsewhere.

This service was built when we split biometric processing out of a single monolithic verification endpoint in mid-2024. The original implementation tried to do everything synchronously in a single request, which caused frequent timeouts and made partial retries impossible.

**Tech stack:** Kotlin / Spring Boot, PostgreSQL (`biometrics_db`).

## Getting Started

### Prerequisites

- JDK 17+
- Docker and Docker Compose (for local PostgreSQL and Kafka)
- Access to document-store-service and customer-service (or enable mocks)

### Running Locally

```bash
# Start infrastructure
docker-compose up -d postgres kafka

# Run the service
./gradlew bootRun --args='--spring.profiles.active=local'
```

To skip external service dependencies during local development:

```yaml
# application-local.yml
document-store-service.mock.enabled: true
customer-service.mock.enabled: true
```

The mock document-store returns a hardcoded S3 reference for testing the capture flow end-to-end.

### Running Tests

```bash
./gradlew test              # Unit tests
./gradlew integrationTest   # Integration tests (requires Docker)
```

Integration tests use Testcontainers for PostgreSQL and an embedded Kafka broker. The biometric session state machine is thoroughly tested -- there are 14 state transition tests covering all happy and failure paths.

## Architecture

The service follows the hexagonal architecture pattern standard across CredityFlow:

- **API layer** -- Spring MVC controllers for capture submission and status queries
- **Domain layer** -- `BiometricSession` (state machine), `CaptureResult`, and session lifecycle logic
- **Infrastructure layer** -- JPA repositories, Kafka producer/consumer, REST clients for document-store and customer-service

The central design element is the `BiometricSession` state machine:

```
INITIATED -> CAPTURED -> LIVENESS_CHECKED -> FACE_MATCHED -> VALIDATED
                                                          -> FAILED
```

State transitions are enforced in the domain layer -- the service rejects any event that would cause an invalid transition (e.g., receiving `face_match_completed` before `liveness_verified`).

### Why a separate orchestrator?

We considered embedding the orchestration logic in the biometrics-capture API itself, but the asynchronous nature of liveness and face-match processing made that impractical. A dedicated session entity with event-driven transitions gives us visibility into where each applicant is in the pipeline and makes retries straightforward.

## API Reference

### POST /api/v1/biometrics/capture

Initiate or update a biometric capture session.

**Request body:**
```json
{
  "proposalId": "proposal-uuid",
  "customerId": "customer-uuid",
  "selfieImageBase64": "...",
  "deviceMetadata": {
    "platform": "ANDROID",
    "osVersion": "14",
    "appVersion": "2.5.1"
  }
}
```

**Response (201):**
```json
{
  "sessionId": "session-uuid",
  "status": "CAPTURED",
  "capturedAt": "2025-08-10T14:22:00Z"
}
```

The selfie image is uploaded to S3 and only the reference is stored in the database. The base64 payload is stripped from memory after upload.

### GET /api/v1/biometrics/{id}/status

Returns the current session status. Used by customer-portal-bff to show verification progress.

**Response (200):**
```json
{
  "sessionId": "session-uuid",
  "status": "VALIDATED",
  "stages": {
    "captured": "2025-08-10T14:22:00Z",
    "livenessChecked": "2025-08-10T14:22:03Z",
    "faceMatched": "2025-08-10T14:22:06Z",
    "validated": "2025-08-10T14:22:06Z"
  }
}
```

## Configuration

| Property | Description | Default |
|---|---|---|
| `spring.datasource.url` | PostgreSQL connection | `jdbc:postgresql://localhost:5432/biometrics_db` |
| `kafka.bootstrap-servers` | Kafka brokers | `localhost:9092` |
| `kafka.topic.biometrics-captured` | Topic for capture events | `biometrics.captured` |
| `kafka.topic.biometrics-validated` | Topic for validation events | `biometrics.validated` |
| `document-store-service.base-url` | Document store URL | `http://document-store-service:8080` |
| `customer-service.base-url` | Customer service URL | `http://customer-service:8080` |
| `s3.bucket.biometrics` | S3 bucket for selfie images | `credityflow-biometrics` |
| `session.stale-timeout-minutes` | Minutes before a session is considered stale | `30` |

## Deployment

Deployed as a container on the CredityFlow Kubernetes cluster (AWS EKS). Helm chart is in `infra/charts/biometrics-capture-service`.

- **Replicas:** 2 in staging, 3 in production
- **Resources:** 768Mi memory, 500m CPU
- **Health checks:** `/actuator/health` (liveness and readiness)
- **Secrets:** Database credentials, S3 access keys, and service-to-service tokens via Kubernetes secrets

The service is stateless from a compute perspective (all state is in PostgreSQL), so horizontal scaling is straightforward.

## Monitoring

### Key Metrics

- `biometric_sessions_created_total` -- Primary volume indicator. Should correlate with document submission rates.
- `biometric_validation_success_rate` -- The percentage of sessions that reach VALIDATED. Currently around 92%.
- `biometric_session_duration_seconds` -- End-to-end time from INITIATED to VALIDATED. P95 target is under 15 seconds.

### Alerts

- **Validation failure rate > 20%** -- Pages on-call. Could indicate liveness-worker issues or a spoofing attack.
- **Session stuck in CAPTURED > 5 minutes** -- Means liveness-worker is not picking up events. Check its health and Kafka consumer lag.
- **Database connection pool > 80%** -- Scale up replicas or investigate slow queries.

### Dashboards

1. **Biometrics Pipeline Overview** -- Sessions by stage (funnel view), success rates, latency percentiles
2. **Capture Quality Metrics** -- Image quality scores, device type breakdown, rejection reasons

## Known Issues

1. **Stale session cleanup is a cron job, not event-driven.** Sessions that get stuck (e.g., liveness-worker never responds) are cleaned up by a scheduled task that runs every 15 minutes. This means a session could sit in a stale state for up to 15 minutes before being marked as FAILED.

2. **No retry for failed S3 uploads.** If the selfie upload to S3 fails, the capture request returns a 500 and the client must retry. There is no server-side retry or queuing.

3. **Base64 image payload is memory-intensive.** Large selfie images (>5MB) can cause memory pressure during peak hours. There is a backlog item to switch to multipart upload, but it has not been prioritized.

4. **PII sensitivity.** This service handles biometric data covered by LGPD. Access to the database and S3 bucket is restricted. Logs are scrubbed of image references. Ensure you have the appropriate access before debugging production issues.
