# Fraud Scoring Service

## Overview

The Fraud Scoring Service is the brain of CredityFlow's fraud prevention pipeline. It collects fraud signals from four upstream sources -- device fingerprinting, watchlist screening, external fraud bureau (ClearCheck), and biometric verification -- aggregates them into a unified feature vector, and runs a machine learning model to produce a fraud risk score from 0 to 1000.

The score itself is not a decision. It feeds into the fraud-decision-service, which applies business rules and thresholds to produce an actionable outcome (approve, deny, or route to manual review). The separation between scoring and decision-making is intentional: the ML model handles the "how risky is this?" question, while business rules handle the "what do we do about it?" question.

**Stack:** Python 3.11 / FastAPI / PostgreSQL (`fraud_scores_db`) / scikit-learn.

The current model is a gradient boosting classifier (GBM) trained on approximately 18 months of labeled fraud data from CredityFlow's loan portfolio. It is retrained quarterly and produces a probability that is scaled to the 0-1000 score range.

## Getting Started

### Prerequisites

- Python 3.11+
- Docker and Docker Compose (PostgreSQL, Kafka)
- The scoring model artifact (`models/fraud_gbm_v4.pkl`)

### Local Setup

```bash
docker-compose up -d postgres kafka
poetry install

# Run database migrations
poetry run alembic upgrade head

# Start the service
poetry run uvicorn fraud_scoring_service.main:app --host 0.0.0.0 --port 8080 --reload
```

For testing the scoring pipeline without waiting for real upstream events, use the signal simulator:

```bash
poetry run python scripts/simulate_signals.py --proposal-id test-001
```

This publishes fake signal events for all four sources, triggering the scoring flow end-to-end.

### Tests

```bash
poetry run pytest tests/unit
poetry run pytest tests/integration
poetry run pytest tests/model_validation  # Accuracy/drift tests
```

The model validation tests load the current model artifact and run it against a holdout dataset, asserting that AUC-ROC remains above 0.92 and that the score distribution has not drifted significantly from the training distribution.

## Architecture

The core design pattern is **signal aggregation with a barrier**:

```
[device_fingerprint_checked] ──┐
[watchlist_checked] ───────────┤
[fraud_bureau_checked] ────────┼──> Signal Aggregator ──> ML Scorer ──> Event Publisher
[biometrics_validated] ────────┘
```

Signals arrive asynchronously. The aggregator tracks which signals have been received for each proposal. Once all four signals are present (or a 60-second timeout fires), the scoring pipeline runs.

### Signal Aggregation

Each proposal has a `FraudSignalAggregate` record in PostgreSQL that tracks:
- Which signals have arrived (bitmask)
- The raw signal data for each source
- Timestamps for arrival order

When the last signal arrives (or the timeout fires), the aggregate transitions to READY and the scoring pipeline is invoked.

### ML Scoring

The scikit-learn GBM model expects a fixed-length feature vector extracted from the signal aggregate. Feature extraction is handled by a set of `FeatureExtractor` classes, one per signal source. The extracted features include:
- Device risk level and specific signal counts
- Watchlist match count and severity
- Bureau fraud signal count and types
- Biometric confidence score and any anomalies

The model outputs a probability (0.0-1.0) that is scaled to a 0-1000 integer score and classified into a risk level.

## API Reference

### GET /api/v1/fraud-scores/{proposal_id}

Retrieve the fraud score for a proposal.

**Response (200):**
```json
{
  "proposalId": "proposal-uuid",
  "score": 342,
  "riskLevel": "MEDIUM",
  "modelVersion": "fraud_gbm_v4",
  "scoredAt": "2025-09-15T11:20:00Z",
  "signalSummary": {
    "deviceRisk": "LOW",
    "watchlistMatches": 0,
    "bureauSignals": 1,
    "biometricConfidence": 0.95
  },
  "signalCompleteness": "COMPLETE"
}
```

If scoring is not yet complete, returns 404.

The `signalCompleteness` field indicates whether all signals arrived (`COMPLETE`) or the timeout fired with missing signals (`PARTIAL`). Partial scores are flagged in the response and tend to route toward manual review downstream.

## Configuration

| Variable | Description | Default |
|---|---|---|
| `DATABASE_URL` | PostgreSQL connection | `postgresql://localhost:5432/fraud_scores_db` |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka brokers | `localhost:9092` |
| `KAFKA_CONSUMER_GROUP` | Consumer group | `fraud-scoring-service` |
| `MODEL_PATH` | Scoring model artifact | `models/fraud_gbm_v4.pkl` |
| `SIGNAL_TIMEOUT_SECONDS` | Max wait for all signals | `60` |
| `SCORE_THRESHOLD_LOW` | Upper bound for LOW risk | `250` |
| `SCORE_THRESHOLD_MEDIUM` | Upper bound for MEDIUM risk | `500` |
| `SCORE_THRESHOLD_HIGH` | Upper bound for HIGH risk | `750` |

## Deployment

Deployed on AWS EKS.

- **Replicas:** 2 in staging, 3 in production
- **Resources:** 1Gi memory, 500m CPU
- **Health checks:** `/health`, `/ready`
- **Model artifact:** Baked into Docker image (~15MB for the GBM pickle)

The 60-second signal timeout is implemented using a scheduled task that checks for stale aggregates every 10 seconds. This means the effective timeout window is 60-70 seconds.

## Monitoring

### Metrics to Watch

- **`fraud_scores_computed_total` by risk_level** -- The distribution should be roughly stable week-over-week. A sudden shift (e.g., 40% of scores jumping to HIGH) suggests either a real fraud wave or a model/data issue.
- **`fraud_signal_timeout_total`** -- Timeouts mean upstream services are slow or down. A few per hour is normal; a sustained increase needs investigation.
- **`fraud_scoring_duration_seconds`** -- The ML inference itself is fast (<50ms). Most latency comes from database reads and feature extraction. P95 should be under 500ms.

### Alerts

- Scoring error rate > 5% -- Something is broken in the pipeline.
- Signal timeout rate > 10% -- An upstream service is likely degraded.
- Average score drifts > 15% from 30-day baseline -- Model drift. Notify the data science team.
- Database connection pool saturation.

### Dashboard

**Fraud Scoring Overview** -- Score distribution histogram, risk level pie chart, signal completeness ratio, throughput over time, scoring latency percentiles.

## Known Issues

1. **Model updates require redeployment.** The scikit-learn model is a pickled artifact baked into the Docker image. There is no hot-reload or model serving infrastructure. The data science team has proposed moving to a model serving platform (MLflow or Seldon), but it has not been prioritized.

2. **Signal timeout can mask upstream failures.** If an upstream service goes down, the scoring service just times out and scores with incomplete data. This is by design (we would rather produce a partial score than block the pipeline), but it means upstream outages may not be immediately visible from the scoring service's perspective.

3. **Feature vector is tightly coupled to upstream schemas.** If device-fingerprint-service changes its signal format, the feature extractor may produce incorrect features without raising an error. Schema versioning for events is on the roadmap.

4. **No A/B testing for model versions.** When a new model is deployed, it replaces the old one for all traffic. Shadow scoring (running both models and comparing) has been discussed but not implemented.

5. **Retraining data extraction is manual.** The data science team runs SQL queries against `fraud_scores_db` to extract training data. There is no automated data pipeline.
