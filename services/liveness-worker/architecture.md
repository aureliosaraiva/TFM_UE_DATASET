# Liveness Worker (svc-11)

**Domain:** Biometrics
**Stack:** Python / FastAPI (worker mode)
**Database:** Redis (ephemeral cache)
**Deployment:** AWS EKS (GPU-enabled node group)

## Component Overview

The Liveness Worker is a specialized anti-spoofing service that determines whether a biometric selfie was captured from a real, physically present human being -- as opposed to a printed photo, a screen replay, or a 3D mask. It is a critical fraud prevention layer in CredityFlow's identity verification pipeline.

This is a queue-driven worker. It does not expose any REST endpoints to other services. It reads from SQS, processes, and writes results to SNS. The only synchronous interface is a health check endpoint used by Kubernetes liveness probes (a fitting name, in this case).

### Internal Architecture

**Anti-Spoofing Model:** The core of the service is a deep learning model (a fine-tuned MobileNetV3) trained to distinguish genuine selfies from presentation attacks. The model analyzes texture patterns, moire effects, edge artifacts, and reflection inconsistencies that are invisible to the human eye but present in photos-of-photos and screen captures. The model runs on GPU for inference, achieving sub-100ms prediction times.

**Depth Analysis Module:** For video-based liveness captures (where the user is asked to turn their head or blink), this module analyzes frame-to-frame depth cues and motion consistency. It uses a lightweight optical flow algorithm to verify natural movement patterns.

**Redis Cache:** Stores recent liveness results keyed by a hash of the image. This prevents redundant processing if the same image is submitted multiple times (which can happen due to mobile app retries on flaky connections). Cache TTL is 10 minutes.

**Result Publisher:** Produces `LivenessCheckCompleted` events to SNS with the verdict (`LIVE`, `SPOOF`, `INCONCLUSIVE`) and a confidence score.

## Data Flow

```
SQS (liveness-check-queue)
        |
        v
  [Check Redis cache] -----> [Cache hit: publish cached result]
        |
        v (cache miss)
  [Download image/video from S3]
        |
        v
  [Anti-Spoofing Model inference]
        |
        v
  [Depth Analysis (if video)]
        |
        v
  [Combine scores, determine verdict]
        |
        v
  [Cache result in Redis]
        |
        v
  SNS: LivenessCheckCompleted
```

The worker runs with a concurrency of 1 message per pod (to avoid GPU contention). KEDA autoscaling monitors the `liveness-check-queue` and scales pods between 1 and 6 replicas based on queue depth.

## Integration Patterns

- **Inbound:** SQS queue fed by the Biometrics Capture Service. Messages contain the S3 key for the selfie/video and the submission ID.
- **Outbound:** SNS event (`LivenessCheckCompleted`) consumed by the Biometrics Capture Service orchestrator.
- **Redis:** Local cluster used only for deduplication caching; not shared with other services.
- **S3:** Read-only access to the biometrics image bucket.

The worker is fully isolated. It shares no state with any other service beyond the message contracts.

## ADR-001: On-Device vs. Server-Side Liveness Detection

**Context:** Many biometric vendors offer on-device (client-side) liveness detection embedded in the mobile SDK. This reduces server load but is vulnerable to SDK tampering on rooted/jailbroken devices. Server-side detection is more secure but adds latency.

**Decision:** Implement server-side liveness detection as the authoritative check. The mobile SDK performs a preliminary client-side check for UX responsiveness (giving the user immediate feedback), but the server-side result is what counts for the official verdict.

**Consequences:**
- Higher security: attackers cannot bypass liveness checks by modifying the client SDK.
- Additional server infrastructure (GPU nodes) required.
- End-to-end latency for biometric verification increases by 2-4 seconds compared to client-only. Acceptable given the one-time nature of the check during onboarding.

## ADR-002: Redis Deduplication Cache

**Context:** In production, approximately 8% of liveness check messages were duplicates caused by mobile app retries and SQS at-least-once delivery. Processing the same image multiple times wasted GPU resources and occasionally produced slightly different confidence scores (due to non-deterministic GPU floating point operations), which confused the orchestrator.

**Decision:** Introduce a Redis cache keyed by SHA-256 hash of the image content. Before processing, the worker checks the cache. If a result exists, it re-publishes the cached result without running inference.

**Consequences:**
- ~8% reduction in GPU compute costs.
- Deterministic results for duplicate submissions.
- Redis is a new infrastructure dependency, but the worker degrades gracefully if Redis is unavailable (it simply processes the image again).
