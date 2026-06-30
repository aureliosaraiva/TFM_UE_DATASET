# Device Fingerprint Service

> **Service name:** `device-fingerprint-service` — part of the Fraud Prevention subdomain.

## Overview

The `device-fingerprint-service` collects and analyzes device-level signals to detect fraud patterns during loan applications. It builds a composite fingerprint from browser characteristics, OS details, screen properties, and other device attributes, then checks this fingerprint against known fraud indicators.

This is one of several services in the Fraud Prevention subdomain. While biometrics answers "is this person who they claim to be?", device fingerprinting answers "is this device suspicious?". Common fraud patterns it catches include device reuse across multiple identities (fraud rings), emulator or automation tool usage, VPN-masked locations, and rooted/jailbroken devices.

The service does not store any personally identifiable information. Device fingerprints are anonymous device-level data points.

**Stack:** Node.js 20 / Express / Redis (`device_fingerprint_cache`).

## Getting Started

### Prerequisites

- Node.js 20+
- Docker and Docker Compose (for Redis and Kafka)

### Running Locally

```bash
docker-compose up -d redis kafka

npm install
npm run dev
```

The dev server starts on port 3000 with hot reload enabled. A seed script populates Redis with a handful of known-bad fingerprints for testing:

```bash
npm run seed:test-data
```

### Tests

```bash
npm test                  # Unit tests (Jest)
npm run test:integration  # Integration tests (needs Docker)
```

## Architecture

The service is a lightweight Express application with three main modules:

- **Fingerprint Parser** -- Accepts raw device data from the client SDK, normalizes it, and computes a canonical fingerprint hash (SHA-256 of sorted, normalized attributes).
- **Signal Detectors** -- A set of pluggable detector functions, each looking for a specific fraud indicator. Currently: `EmulatorDetector`, `VpnDetector`, `RootedDeviceDetector`, `AutomationToolDetector`, `DeviceReuseDetector`, `GeoMismatchDetector`.
- **Risk Scorer** -- Aggregates detected signals into a risk level (LOW, MEDIUM, HIGH) using a weighted scoring formula.

Redis is the sole data store. There is no relational database. Fingerprints are stored as Redis hashes with a 90-day TTL. The device-to-identity mapping (for cross-identity reuse detection) is maintained as Redis sets keyed by fingerprint hash.

### Why Redis-Only?

Device fingerprint lookups need to be fast (sub-10ms). The data is ephemeral by nature -- a fingerprint from 90 days ago has limited fraud value. Redis gives us the speed and TTL semantics we need without the operational overhead of a relational database. The tradeoff is durability: if Redis is flushed, we lose the fingerprint history. In practice, this has happened once (a misconfigured `FLUSHALL` in staging) and was annoying but not catastrophic -- the fingerprint database rebuilds organically as new applications flow through.

## API Reference

### POST /api/v1/fingerprint/analyze

Submit a device fingerprint for analysis.

**Request:**
```json
{
  "proposalId": "proposal-uuid",
  "userAgent": "Mozilla/5.0 ...",
  "screenResolution": "1920x1080",
  "timezone": "America/Sao_Paulo",
  "language": "pt-BR",
  "platform": "Linux x86_64",
  "canvasHash": "a1b2c3d4...",
  "webglVendor": "Google Inc. (NVIDIA)",
  "installedPlugins": ["PDF Viewer"],
  "touchSupport": false,
  "ip": "177.45.12.88",
  "declaredState": "SP"
}
```

**Response (200):**
```json
{
  "fingerprintId": "fp-hash-uuid",
  "riskLevel": "MEDIUM",
  "signals": [
    {"type": "VPN_DETECTED", "confidence": 0.78},
    {"type": "DEVICE_REUSE", "confidence": 1.0, "previousIdentities": 2}
  ],
  "seenBefore": true,
  "firstSeenAt": "2025-07-01T08:00:00Z"
}
```

### GET /api/v1/fingerprint/{id}

Retrieve a previously analyzed fingerprint and its signals.

## Configuration

| Variable | Description | Default |
|---|---|---|
| `PORT` | HTTP server port | `3000` |
| `REDIS_URL` | Redis connection | `redis://localhost:6379/1` |
| `KAFKA_BROKERS` | Kafka brokers | `localhost:9092` |
| `KAFKA_TOPIC_INPUT` | Input topic | `biometrics.validated` |
| `KAFKA_TOPIC_OUTPUT` | Output topic | `fraud.device_fingerprint.checked` |
| `FINGERPRINT_TTL_DAYS` | TTL for stored fingerprints | `90` |
| `DEVICE_REUSE_THRESHOLD` | Identities per device to flag | `2` |
| `GEO_IP_DB_PATH` | MaxMind GeoIP database path | `data/GeoLite2-City.mmdb` |

## Deployment

Deployed on AWS EKS. Lightweight footprint.

- **Replicas:** 2 in staging, 3 in production
- **Resources:** 256Mi memory, 250m CPU (Node.js is efficient for this workload)
- **Health checks:** `GET /health`
- **Scaling:** HPA based on CPU utilization (threshold: 70%)

The MaxMind GeoIP database is updated weekly via a CronJob that downloads the latest database and triggers a rolling restart.

## Monitoring

### Key Metrics

- `device_fingerprints_analyzed_total` -- Overall volume. Should correlate with biometrics validation volume.
- `device_risk_level_total` by level -- Distribution of risk assessments. Typical split: ~80% LOW, ~15% MEDIUM, ~5% HIGH.
- `device_reuse_detected_total` -- A sharp increase may indicate a fraud ring. The fraud team monitors this closely.

### Alerts

- Device reuse rate > 15% over 30 minutes -- Possible fraud ring. Notifies the fraud team Slack channel.
- Emulator detection > 10% over 1 hour -- Unusual. Typically under 2%.
- Redis connection failure -- Pages on-call.
- Analysis error rate > 5% -- Usually a Redis issue.

### Dashboard

**Device Fingerprint Overview** -- Volume, risk distribution pie chart, signal breakdown bar chart, device reuse trend line, top reused fingerprints table.

## Known Issues

1. **Browser privacy features reduce fingerprint accuracy.** Browsers like Brave and Firefox with strict privacy settings randomize canvas hashes and block WebGL queries. This makes fingerprints less unique and increases the chance that two different devices produce the same hash. We compensate by giving less weight to these attributes when they appear randomized.

2. **GeoIP accuracy is limited.** The MaxMind database has city-level accuracy for about 80% of Brazilian IPs. Some ISPs route traffic through regional hubs, making the detected location different from the actual location. The `GEO_MISMATCH` signal has a higher false positive rate than other signals.

3. **No durable storage.** If Redis data is lost, historical fingerprint data is gone. The team has discussed adding a PostgreSQL write-behind for long-term analysis, but it has not been prioritized since the real-time fraud detection use case is the primary concern.

4. **Shared/public devices.** Internet cafes, libraries, and shared family devices generate false positives for `DEVICE_REUSE`. The risk scorer gives this signal lower weight when the device type suggests a public terminal, but it is not a perfect heuristic.
