# Biometrics Capture Service (svc-10)

**Domain:** Biometrics
**Stack:** Kotlin / Spring Boot
**Database:** PostgreSQL (`biometrics_db`)
**Deployment:** AWS EKS

## Component Overview

The Biometrics Capture Service acts as the orchestrator for CredityFlow's biometric verification sub-flow. It does not perform liveness detection or face matching directly -- those capabilities live in dedicated worker services (svc-11 and svc-12). Instead, this service coordinates the end-to-end biometric verification process: receiving selfie captures from the mobile app, dispatching them to the appropriate worker, aggregating results, and rendering a biometric verification verdict.

### Architecture Breakdown

**Capture Endpoint:** Accepts selfie images and short video clips from the mobile SDK. The payload includes device metadata (OS version, camera specs, SDK version) that feeds into the anti-spoofing pipeline. Images are stored temporarily in S3 and referenced by ID throughout the sub-flow.

**Orchestration Engine:** Manages a lightweight two-step workflow per biometric submission:
1. Send the selfie/video to the Liveness Worker (svc-11) for anti-spoofing analysis.
2. If liveness passes, send the selfie to the Face Match Service (svc-12) for comparison against the customer's identity document photo.

Both steps are dispatched asynchronously via SQS. The orchestrator tracks the state of each submission in PostgreSQL and reacts to completion events.

**Result Aggregator:** Combines liveness and face match results into a single biometric verdict (`VERIFIED`, `FAILED`, `MANUAL_REVIEW`). The verdict is persisted and published as a `BiometricsVerified` or `BiometricsFailed` event.

**Retry Manager:** If a worker fails to process within the timeout window (90 seconds), the orchestrator retries once with exponential backoff. After two failures, the submission is routed to manual review.

## Data Flow

1. Mobile app submits a selfie capture to the Capture Endpoint.
2. The image is stored in S3; a `biometric_submissions` record is created in PostgreSQL with status `PROCESSING`.
3. A liveness check request is published to the `liveness-check-queue` (SQS).
4. Liveness Worker processes the image and publishes a `LivenessCheckCompleted` event.
5. Orchestration Engine receives the event. If liveness passed, it publishes a face match request to the `face-match-queue`.
6. Face Match Service compares the selfie against the document photo and publishes `FaceMatchCompleted`.
7. Result Aggregator computes the final verdict and publishes `BiometricsVerified` or `BiometricsFailed` to SNS.
8. Proposal Service consumes the event and advances the proposal state.

## Integration Patterns

- **Mobile SDK -> REST:** Selfie capture upload endpoint, designed for unreliable mobile networks (supports chunked upload, resume on failure).
- **SQS dispatch:** Liveness and face match requests are dispatched via SQS for decoupled, async processing.
- **SNS consumption:** Listens for `LivenessCheckCompleted` and `FaceMatchCompleted` events from workers.
- **SNS publication:** Emits `BiometricsVerified` / `BiometricsFailed` events consumed by Proposal Service and Fraud Decision Service.

## ADR-001: Orchestrator Pattern for Biometric Sub-Flow

**Context:** The biometric verification involves two sequential steps (liveness, then face match) with error handling, retries, and result aggregation. The team debated whether to implement this as pure choreography (each worker triggers the next) or as an explicit orchestrator.

**Decision:** Use a lightweight orchestrator within this service. Unlike the broader proposal lifecycle (which uses choreography), the biometric sub-flow is tightly coupled, has a fixed sequence, and requires coordinated error handling. An orchestrator is the better fit here.

**Consequences:**
- Clear visibility into the biometric verification state for debugging and monitoring.
- The orchestrator becomes a single point of failure for the biometric flow; mitigated by running 3+ replicas with leader election for retry processing.
- Adding a third step (e.g., voice verification) would require changes to the orchestrator, but this is acceptable given the low rate of change.

## ADR-002: Temporary S3 Storage for Biometric Images

**Context:** Biometric images contain highly sensitive PII (face photos). The team evaluated storing them in PostgreSQL (as bytea), in a dedicated encrypted S3 bucket, or not storing them at all (process in memory and discard).

**Decision:** Store images temporarily in a dedicated S3 bucket with a 7-day lifecycle policy. Images are encrypted with a KMS key specific to the biometrics domain. After 7 days, they are automatically deleted. If an audit requires the image, the verdict record in PostgreSQL contains enough metadata (confidence scores, timestamps, device info) to reconstruct the decision context.

**Consequences:**
- Minimized PII retention: biometric images exist only as long as needed for processing and short-term dispute resolution.
- Regulatory compliance with LGPD's data minimization principle.
- If a customer disputes a biometric decision after 7 days, the team must rely on metadata rather than the original image. This tradeoff was accepted by legal.
