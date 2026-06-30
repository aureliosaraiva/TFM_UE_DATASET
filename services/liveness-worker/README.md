# Liveness Worker

## Overview

A background worker that runs liveness detection (anti-spoofing) on selfie images captured during CredityFlow's biometric verification flow. Its job is simple but critical: determine whether a selfie was taken from a real, live person or from a photo, screen, or mask.

This is a pure event-driven worker with no HTTP API. It listens for `biometrics_captured` events, downloads the selfie from S3, runs the liveness model, and publishes a `liveness_verified` event with the result. The whole thing usually takes 2-4 seconds per image.

**Stack:** Python 3.11 / FastAPI (for internal health endpoint only) / Redis (`biometrics_cache`).

The liveness model is a custom ensemble that combines texture analysis (LBP features) with a lightweight depth estimation network. It was trained on a dataset of ~50k real and spoofed images collected from CredityFlow's own user base (with consent) and augmented with public anti-spoofing datasets.

## Getting Started

### Prerequisites

- Python 3.11+
- Docker and Docker Compose (Redis, Kafka)
- The liveness model artifact (`models/liveness_v3.pkl`) -- pulled from the model registry during Docker build

### Running Locally

```bash
# Start dependencies
docker-compose up -d redis kafka

# Install packages
poetry install

# Run the worker
poetry run python -m liveness_worker.main --profile local
```

The worker will start consuming from the `biometrics.captured` Kafka topic. For local testing, you can publish a test event:

```bash
poetry run python scripts/publish_test_event.py --image-path test_data/sample_selfie.jpg
```

### Running Tests

```bash
poetry run pytest tests/unit           # Unit tests (~2s)
poetry run pytest tests/integration    # Integration tests (~30s, needs Docker)
```

The integration tests spin up Redis via Testcontainers and use a mock S3 (moto) for image downloads.

## Architecture

The worker is structured as a simple pipeline:

```
Kafka Consumer -> Image Downloader -> Liveness Analyzer -> Result Publisher
```

- **Kafka Consumer** -- Reads `biometrics_captured` events. Uses manual offset commit after successful processing.
- **Image Downloader** -- Fetches the selfie from S3. Images are held in memory only for the duration of analysis.
- **Liveness Analyzer** -- Runs the ML model. Outputs a confidence score (0.0-1.0) and a list of spoofing signals.
- **Result Publisher** -- Writes the result to Redis (for deduplication) and publishes `liveness_verified` to Kafka.

The Redis cache serves two purposes: deduplication (if the same image is processed twice due to a Kafka retry, the cached result is returned instead of re-running the model) and temporary storage of results for debugging.

There is no HTTP API for external consumers. FastAPI is used solely for the `/health` and `/ready` endpoints required by Kubernetes.

## API Reference

No external API. This is a worker service.

Internal endpoints (not for service-to-service communication):

| Endpoint | Purpose |
|---|---|
| `GET /health` | Kubernetes liveness probe |
| `GET /ready` | Kubernetes readiness probe (checks Kafka and Redis connectivity) |

## Configuration

| Variable | Description | Default |
|---|---|---|
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka broker addresses | `localhost:9092` |
| `KAFKA_CONSUMER_GROUP` | Consumer group ID | `liveness-worker` |
| `KAFKA_TOPIC_INPUT` | Input topic | `biometrics.captured` |
| `KAFKA_TOPIC_OUTPUT` | Output topic | `biometrics.liveness.verified` |
| `REDIS_URL` | Redis connection string | `redis://localhost:6379/0` |
| `REDIS_RESULT_TTL_SECONDS` | TTL for cached results | `3600` |
| `S3_BUCKET_BIOMETRICS` | S3 bucket for selfie images | `credityflow-biometrics` |
| `MODEL_PATH` | Path to liveness model artifact | `models/liveness_v3.pkl` |
| `LIVENESS_THRESHOLD` | Minimum confidence to pass | `0.85` |
| `MAX_CONCURRENT_ANALYSES` | Parallel processing limit | `4` |

## Deployment

Deployed on AWS EKS as a Kubernetes Deployment (not a Job -- it runs continuously).

- **Replicas:** 2 in staging, 4 in production
- **Resources:** 1Gi memory, 1 CPU (model inference is CPU-bound)
- **Health checks:** `/health` (liveness), `/ready` (readiness)
- **Scaling:** HPA based on Kafka consumer lag. Scales up when lag exceeds 50 messages.

The model artifact is baked into the Docker image at build time. Model updates require a new image build and rolling deployment. There is no hot-reload.

## Monitoring

### What to Watch

- **`liveness_checks_total`** -- Should closely follow `biometrics_captured` event volume. If it lags, check consumer lag.
- **`liveness_check_duration_seconds`** -- P95 should be under 5 seconds. If it climbs, check CPU throttling.
- **`spoofing_signals_detected_total`** -- A sudden spike in any signal type warrants investigation. The `SCREEN_REFLECTION` signal occasionally produces false positives on glossy phone screens.
- **`liveness_cache_hit_total`** -- Cache hits mean Kafka retries are happening. A few per hour is normal; a flood indicates upstream issues.

### Alerts

- Liveness check error rate > 10% for 5 minutes
- P99 latency above 8 seconds
- Spoof detection rate > 40% over 30 minutes (coordinated spoofing attempt or model issue)
- Redis connection lost

### Dashboard

**Liveness Worker Health** -- Throughput, latency histogram, error rate, spoofing signal breakdown by type, consumer lag.

## Known Issues

1. **Model accuracy is approximately 97%.** About 3% of legitimate selfies get flagged as spoofing, particularly low-light images and certain phone models with aggressive beauty filters. The team is collecting false positive samples for the next model iteration.

2. **No dead-letter queue.** If an image fails to process (corrupt file, S3 timeout), the Kafka message is retried up to 3 times and then skipped. These failures require manual investigation via the application logs.

3. **Memory spikes with large images.** High-resolution selfies (>10MP) can cause memory spikes during model inference. The image is downscaled to 640x480 before analysis, but the original is held in memory until downscaling completes. A streaming resize would help but has not been implemented.

4. **PII handling.** Selfie images are downloaded to memory, processed, and immediately discarded. No images are written to disk or persisted by this service. However, during debugging, be aware that error logs may contain S3 keys that could be used to retrieve the original image.
