# credit-scoring-engine

`svc-20` -- Credit Analysis -- team-credit

## Overview

The credit-scoring-engine is a Python/FastAPI service that computes credit risk scores for loan proposals. It uses a trained XGBoost model to evaluate a feature vector assembled from bureau data, verified income, and fraud signals. Scores range from 0 to 1000, where higher values indicate lower risk.

The service implements the **CQRS pattern**:
- **Write side:** Kafka consumers ingest domain events, build feature vectors, and run model inference. Results are persisted to PostgreSQL.
- **Read side:** A FastAPI REST endpoint serves pre-computed scores by `proposal_id`.

This separation keeps the write path optimised for throughput (batch-friendly, async) and the read path optimised for latency (simple indexed lookups).

## Getting Started

### Requirements

- Python 3.11+
- PostgreSQL 15+
- Kafka broker
- Model artefact file (`model.xgb`) — download from the ML artefact registry

### Local Setup

```bash
# Create virtual environment
python -m venv .venv && source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Configure
cp .env.example .env
# Edit .env with your local database and Kafka settings

# Run database migrations
alembic upgrade head

# Start the service
uvicorn app.main:app --reload --port 8000
```

### Testing

```bash
pytest                           # all tests
pytest tests/unit/               # unit tests only
pytest tests/integration/        # integration tests (needs Postgres + Kafka)
pytest --cov=app --cov-report=html  # coverage report
```

## Architecture

```
                 Events (Kafka)                    REST (FastAPI)
                      |                                  |
         +------------+------------+          +----------+----------+
         |        Write Side       |          |      Read Side      |
         |                         |          |                     |
         | - Event consumers       |          | - GET /credit-scores|
         | - Feature assembly      |          | - DB lookup         |
         | - Model inference       |          |                     |
         | - Score persistence     |          |                     |
         +------------+------------+          +----------+----------+
                      |                                  |
                      +---------> PostgreSQL <------------+
```

The write side waits for all three input events (`bureau_data_fetched`, `income_verified`, `fraud_cleared`) before triggering inference. A readiness gate tracks which features have arrived for each proposal.

Model versioning is supported: the active model version is configurable, and shadow scoring can run a candidate model in parallel without publishing results.

See [architecture.md](./architecture.md) for detailed diagrams and architectural decision records.

## API Reference

### GET /api/v1/credit-scores/{proposal_id}

Retrieves the computed credit score for a proposal.

**Response (200):**
```json
{
  "proposal_id": "550e8400-e29b-41d4-a716-446655440000",
  "score": 742,
  "risk_tier": "LOW",
  "model_version": "v3.2.1",
  "top_factors": [
    {"feature": "bureau_score", "weight": 0.31},
    {"feature": "debt_to_income", "weight": 0.22},
    {"feature": "credit_history_length", "weight": 0.15}
  ],
  "computed_at": "2026-03-29T09:15:00Z"
}
```

**Response (404):** Score not yet computed for this proposal.

## Configuration

Environment variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | — (required) |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka broker addresses | `localhost:9092` |
| `KAFKA_CONSUMER_GROUP` | Consumer group ID | `credit-scoring-cg` |
| `MODEL_PATH` | Path to the XGBoost model file | `./models/model.xgb` |
| `ACTIVE_MODEL_VERSION` | Model version to use for scoring | `latest` |
| `SHADOW_MODEL_PATH` | Path to shadow model (optional) | — |
| `FEATURE_READINESS_TIMEOUT_S` | Max wait for all features (seconds) | `300` |
| `LOG_LEVEL` | Logging level | `INFO` |

## Deployment

Containerised with Docker and deployed on AWS EKS.

```bash
# Build
docker build -t credit-scoring-engine:latest .

# Deploy
helm upgrade --install credit-scoring-engine ./charts/credit-scoring-engine \
  --namespace credit-analysis \
  --set image.tag=$(git rev-parse --short HEAD)
```

**Resources:** 1 Gi memory, 500m CPU (request); 2 Gi memory, 1 CPU (limit). Model inference is CPU-bound — consider dedicated node pools for production.

**Model artefacts** are pulled from S3 at container startup. The `MODEL_PATH` environment variable points to the local copy.

## Monitoring

| Dashboard | Purpose |
|-----------|---------|
| Credit Scoring Overview | Volume, score distribution, latency |
| Model Performance | AUC trends, score drift, feature importance |
| Pipeline Health | Feature completeness, event lag per proposal |

**Critical alerts:**
- `scoring_error_rate_high` — inference errors > 1% over 5 min
- `scoring_latency_p99_high` — p99 > 2s
- `score_distribution_drift` — mean score shifts > 50 points week-over-week

## Known Issues

1. **Feature readiness timeout** — If one of the three input events never arrives (e.g., income verification fails), the proposal will sit in the readiness gate until the timeout expires (`FEATURE_READINESS_TIMEOUT_S`). After timeout, the service publishes a `credit_scored` event with a fallback conservative score. This behaviour needs review — see CRED-2250.

2. **Cold start model loading** — The XGBoost model file (~200MB) is loaded into memory at startup. First inference after a pod restart takes 3-5s longer than steady state. The readiness probe accounts for this, but rapid scaling events can cause brief latency spikes.

3. **Shadow scoring resource overhead** — Running a shadow model doubles CPU usage during inference. Shadow scoring should be disabled in resource-constrained environments or during traffic peaks.
