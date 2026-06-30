# ocr-worker

## Overview

The ocr-worker is a stateless background processor that extracts text from uploaded documents. It consumes `documents_uploaded` events, fetches the corresponding file from S3, runs it through Tesseract OCR and custom-trained extraction models, and publishes the results for downstream classification.

Unlike most CredityFlow services, the ocr-worker is written in **Python** using **FastAPI** (for health/readiness probes only -- it has no user-facing REST API). It has no database; all state lives in the messages it consumes and produces.

**Team**: team-documents | **Domain**: Document Management

## Getting Started

### Prerequisites
- Python 3.11+
- Tesseract 5.x installed (`brew install tesseract` on macOS)
- Portuguese language pack for Tesseract (`por.traineddata`)
- Docker Compose (for LocalStack S3 and SQS)

### Local Development

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Start S3 and SQS via LocalStack
docker-compose up -d localstack

# Run the worker
python -m ocr_worker.main --profile local
```

The worker polls the SQS queue and processes messages one at a time in local mode. To simulate a document upload, push a test message to the local SQS queue using the AWS CLI.

### Tests

```bash
pytest tests/unit               # Fast, no external deps
pytest tests/integration        # Requires Tesseract installed
pytest tests/e2e                # Requires Docker + LocalStack
```

## Architecture

The worker follows a simple pipeline pattern:

1. **Poll** SQS for `documents_uploaded` messages
2. **Fetch** the file from S3
3. **Pre-process** the image (deskew, contrast, denoise)
4. **Extract** text using Tesseract
5. **Structured extraction** using custom models for known field patterns
6. **Publish** results to SNS

There is no REST API beyond health probes at `/health` and `/ready`. The worker is designed to scale horizontally by increasing replica count -- each instance competes for messages on the same SQS queue.

Custom models are stored in S3 and loaded on startup. Model versioning is tracked in the published event payload so downstream consumers know which model produced the output.

See [architecture.md](architecture.md) for data flow details.

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `SQS_QUEUE_URL` | SQS queue for documents_uploaded events | - |
| `SNS_TOPIC_ARN` | SNS topic for OCR result events | - |
| `S3_BUCKET` | Bucket where document files are stored | - |
| `TESSERACT_LANG` | Tesseract language pack | `por` |
| `MODEL_S3_PATH` | S3 path to custom extraction models | - |
| `MAX_RETRIES` | Retries for low-confidence extractions | `3` |
| `CONFIDENCE_THRESHOLD` | Minimum acceptable confidence (0-1) | `0.7` |

## Deployment

Runs on EKS in a dedicated node pool with higher CPU allocation, since OCR is compute-intensive.

- **CPU**: 1000m request / 2000m limit
- **Memory**: 1Gi request / 2Gi limit
- **Replicas**: 3 (staging), 6 (production)
- **HPA**: scales on SQS queue depth (target: < 100 messages)

The Docker image is ~1.2GB due to Tesseract and model files. A multi-stage build keeps the final image as lean as possible.

## Monitoring

Key metrics (exposed via Prometheus client):
- `ocr_documents_processed_total` (by status: success, failed, retry)
- `ocr_processing_duration_seconds` (p50, p95, p99)
- `ocr_field_confidence_score` (by field name)
- `sqs_messages_in_flight`

Alerts:
- Failure rate > 10% in a 1-hour window
- p95 latency > 30 seconds
- Queue depth growing continuously for > 15 minutes
- Pod crash loops

Dashboard: "OCR Worker Health" in Grafana shows throughput, latency percentiles, and confidence score distributions by document type.

## Known Issues

- **Low-quality scans**: Photos taken in poor lighting or at an angle produce low confidence scores. The pre-processing pipeline helps but cannot fully compensate. Customers sometimes need to re-upload.
- **Memory spikes**: Very large PDFs (>20 pages) can cause OOM kills. A page-limit check was added but edge cases remain.
- **Single language**: Only Portuguese (`por`) is supported. Documents in English or Spanish are not handled.
- **Model retraining**: Custom models need manual retraining when document issuers change their formats. There is no automated drift detection.
