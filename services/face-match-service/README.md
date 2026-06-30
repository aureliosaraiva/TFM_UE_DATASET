# Face Match Service

## Overview

This service performs the final step in CredityFlow's biometric identity verification: comparing the applicant's live selfie against the photo on their identity document (RG or CNH). It uses facial embedding comparison to produce a similarity score and a pass/fail result.

The face-match-service sits at the end of the biometric pipeline. By the time it runs, the selfie has already passed liveness detection (liveness-worker confirmed it is a live person, not a spoofed image). The question this service answers is: "Is this live person the same person on the ID document?"

**Stack:** Python 3.11 / FastAPI / PostgreSQL (`biometrics_db`).

The facial embedding model is a ResNet-based architecture fine-tuned on Brazilian ID documents. It produces a 512-dimensional embedding vector for each face, and the comparison is a straightforward cosine similarity calculation.

## Getting Started

### Prerequisites

- Python 3.11+
- Docker and Docker Compose (PostgreSQL, Kafka, Redis)
- The face embedding model artifact (`models/face_embed_v2.onnx`)

### Running Locally

```bash
docker-compose up -d postgres kafka
poetry install
poetry run uvicorn face_match_service.main:app --host 0.0.0.0 --port 8080 --reload
```

For testing without real images, enable the mock embedding generator:

```bash
export FACE_MATCH_MOCK_EMBEDDINGS=true
```

This returns deterministic embeddings based on the file hash, useful for integration test reproducibility.

### Tests

```bash
poetry run pytest tests/unit
poetry run pytest tests/integration    # Needs Docker
poetry run pytest tests/model          # Model accuracy tests against labeled dataset
```

The model accuracy tests run the embedding model against a labeled dataset of 500 image pairs and assert that precision and recall remain above configured thresholds. These tests take about 2 minutes.

## Architecture

```
Event Consumer / REST API
        |
        v
  ImageRetriever (S3 + document-store-service)
        |
        v
  EmbeddingGenerator (ONNX Runtime)
        |
        v
  SimilarityCalculator (cosine similarity + quality adjustments)
        |
        v
  ResultPersistence (PostgreSQL) + Event Publisher (Kafka)
```

The service has two entry points: the Kafka consumer for the normal event-driven flow and a REST endpoint for on-demand matches (used by backoffice tooling and testing).

Image retrieval involves two steps: first, calling document-store-service to get the document image S3 reference, then downloading both the selfie and document photo from S3. Both downloads happen in parallel.

### Quality Adjustments

Raw cosine similarity is adjusted based on image quality factors:

- **Lighting** -- Dark or overexposed images get a quality penalty
- **Resolution** -- Low-resolution document photos (common with older IDs) reduce confidence
- **Face angle** -- Non-frontal faces reduce accuracy; a small angle correction is applied

The adjustment never increases the score, only decreases it. This makes the system conservative -- if the image quality is poor, the match threshold becomes harder to meet, which may route the application to manual review rather than auto-approving on a shaky match.

## API Reference

### POST /api/v1/face-match

Internal endpoint. Not exposed through the API gateway.

**Request:**
```json
{
  "sessionId": "session-uuid",
  "selfieImageRef": "s3://credityflow-biometrics/selfies/abc123.jpg",
  "documentImageRef": "s3://credityflow-documents/ids/def456.jpg"
}
```

**Response (200):**
```json
{
  "sessionId": "session-uuid",
  "result": "MATCH",
  "similarityScore": 0.94,
  "threshold": 0.80,
  "qualityAdjustments": {
    "lighting": -0.02,
    "resolution": 0.00,
    "faceAngle": -0.01
  },
  "modelVersion": "face_embed_v2"
}
```

Possible `result` values: `MATCH`, `NO_MATCH`, `INCONCLUSIVE` (when the adjusted score is within 5% of the threshold).

## Configuration

| Variable | Description | Default |
|---|---|---|
| `DATABASE_URL` | PostgreSQL connection | `postgresql://localhost:5432/biometrics_db` |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka brokers | `localhost:9092` |
| `KAFKA_TOPIC_INPUT` | Input topic | `biometrics.liveness.verified` |
| `KAFKA_TOPIC_OUTPUT` | Output topic | `biometrics.face_match.completed` |
| `S3_BUCKET_BIOMETRICS` | Selfie image bucket | `credityflow-biometrics` |
| `DOCUMENT_STORE_BASE_URL` | Document store URL | `http://document-store-service:8080` |
| `MODEL_PATH` | Embedding model path | `models/face_embed_v2.onnx` |
| `MATCH_THRESHOLD` | Similarity threshold | `0.80` |
| `INCONCLUSIVE_MARGIN` | Margin for INCONCLUSIVE | `0.05` |

## Deployment

Deployed on AWS EKS via Helm.

- **Replicas:** 2 in staging, 3 in production
- **Resources:** 1.5Gi memory, 1 CPU (embedding inference is compute-heavy)
- **Health checks:** `/health` and `/ready`
- **Model artifact:** Baked into the Docker image. The ONNX model is ~90MB.

The service uses ONNX Runtime for inference, which provides better performance than raw PyTorch for serving. CPU inference takes ~200ms per image pair.

## Monitoring

### Metrics

- **`face_match_total` by result** -- The match/no-match/inconclusive ratio. Historically about 88% match, 8% no-match, 4% inconclusive.
- **`face_match_confidence_score`** -- Distribution of similarity scores. The histogram should show a clear bimodal distribution (genuine matches clustered around 0.90-0.98, non-matches around 0.30-0.50). If the modes start overlapping, the model may need retraining.
- **`face_match_duration_seconds`** -- P95 should be under 3 seconds including image download.

### Alerts

- Match failure rate > 25% for 15 minutes
- Document photo retrieval errors > 5%
- Average confidence score drops by more than 10% from baseline

### Dashboard

**Face Match Pipeline** -- Match rates, confidence score histogram, latency percentiles, quality adjustment distribution.

## Known Issues

1. **Old/worn ID documents cause low-quality matches.** Document photos on older RG cards can be faded or scratched, reducing embedding quality. The quality adjustment helps, but some legitimate applicants end up with INCONCLUSIVE results and must be reviewed manually.

2. **Single global threshold.** The 0.80 similarity threshold applies to all document types. RG photos tend to be lower quality than CNH photos, so the false rejection rate is higher for RG-based applications. Per-document-type thresholds are planned.

3. **No model A/B testing.** Deploying a new embedding model is all-or-nothing. The team wants to add shadow scoring (run new model in parallel, compare results, but use old model for decisions) but the infrastructure is not in place yet.

4. **PII.** Facial embeddings are stored in the database. While an embedding cannot be reverse-engineered into a face image, it is still considered biometric PII under LGPD.
