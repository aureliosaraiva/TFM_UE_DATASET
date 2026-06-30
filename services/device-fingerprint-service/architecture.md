# Device Fingerprint Service (svc-13)

**Domain:** Fraud
**Stack:** Node.js / Express
**Database:** Redis
**Deployment:** AWS EKS

## Component Overview

The Device Fingerprint Service collects and analyzes browser and device signals to create a unique fingerprint for each device used during the CredityFlow onboarding process. This fingerprint serves as one of three input signals to the downstream Fraud Scoring Service, helping identify devices associated with fraudulent activity.

This is one of only two Node.js services in the CredityFlow ecosystem (the other being an internal BFF). Node.js was chosen here because the service needs to execute JavaScript-based fingerprinting logic that mirrors the client-side collection SDK, and because the workload is I/O-bound with high concurrency requirements.

### Architectural Components

**Collection Endpoint:** A REST endpoint that receives raw device signals from the frontend JavaScript SDK. Signals include: user agent string, screen resolution, installed fonts, WebGL renderer, timezone, language preferences, canvas fingerprint hash, audio context fingerprint, battery status, installed plugins, and approximately 30 additional browser attributes. The payload is typically 2-4 KB.

**Fingerprint Generator:** Normalizes and hashes the collected signals into a deterministic device fingerprint (a 64-character hex string). The algorithm accounts for minor variations (e.g., browser version updates) by using a fuzzy matching approach where the fingerprint is computed from the most stable subset of signals.

**Device History (Redis):** Each fingerprint is stored in Redis with an associated set of metadata: first seen timestamp, last seen timestamp, number of onboarding attempts, associated customer IDs, and risk flags. Redis Sorted Sets are used to track frequency patterns (e.g., how many distinct customers have used this device in the last 30 days). TTL is 90 days.

**Risk Signal Publisher:** After computing the fingerprint and checking device history, the service publishes a `DeviceFingerprinted` event to SNS. The event includes the fingerprint, a device risk score (0-100), and flagged anomalies (e.g., "device seen with 5+ different identities," "known emulator detected," "VPN/proxy detected").

## Data Flow

1. The frontend SDK (a lightweight JavaScript snippet) collects device signals on the client side and posts them to the Collection Endpoint.
2. The service generates the fingerprint and queries Redis for historical data.
3. Device risk signals are computed:
   - New device = neutral signal.
   - Device seen with 1-2 identities = low risk.
   - Device seen with 3+ identities in 30 days = elevated risk.
   - Known emulator or automation tool signatures = high risk.
4. The fingerprint, risk score, and metadata are stored/updated in Redis.
5. A `DeviceFingerprinted` event is published to SNS for the Fraud Scoring Service.

## Integration Patterns

- **REST (inbound):** The frontend SDK posts device signals. This endpoint is public-facing (behind API Gateway with rate limiting and WAF).
- **SNS (outbound):** `DeviceFingerprinted` events consumed by `fraud-scoring-service`.
- **Redis:** Used as the sole data store. No PostgreSQL. Device data is inherently ephemeral and the access patterns (high-frequency reads/writes on individual keys) are ideal for Redis.
- **No inbound async:** This service does not consume any SQS queues. It is triggered synchronously by the frontend.

## ADR-001: Redis as Primary Store (No PostgreSQL)

**Context:** The platform standard is PostgreSQL for persistent data. However, the device fingerprint data has characteristics that make Redis a better fit: high write frequency (every page load), key-value access patterns, natural TTL (90 days), and no need for complex queries or joins.

**Decision:** Use Redis (ElastiCache) as the sole data store. No PostgreSQL instance. Data older than 90 days is automatically evicted via TTL.

**Consequences:**
- Sub-millisecond read/write latency for fingerprint lookups during the critical onboarding path.
- Data is not durable in the traditional sense; a Redis failure could lose recent fingerprint data. Mitigated by ElastiCache Multi-AZ with automatic failover.
- No relational queries possible; if the fraud team needs to run analytics on device data, an export job periodically dumps Redis data to S3 for Athena queries.

## ADR-002: Node.js for JavaScript Signal Parity

**Context:** The device fingerprinting algorithm must produce identical results when run server-side (for validation) as when run client-side (for collection). The client SDK is written in JavaScript. Reimplementing the fingerprint hash algorithm in Kotlin or Python introduced subtle inconsistencies due to differences in Unicode handling and hashing libraries.

**Decision:** Implement the server-side service in Node.js so the exact same fingerprinting code can be shared between client and server as an npm package.

**Consequences:**
- Perfect parity between client-side and server-side fingerprint computation.
- The team must maintain a Node.js service in an otherwise Kotlin/Python ecosystem. Mitigated by keeping the service small and focused.
- Node.js's event-driven I/O model is well-suited to this service's workload (high concurrency, I/O-bound, no CPU-intensive computation on the main thread).
