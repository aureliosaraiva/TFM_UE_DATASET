# document-classifier-service

## Overview

After a document has been through OCR, the next step is figuring out *what kind* of document it is. That is the job of the document-classifier-service. It takes the extracted text and fields from the ocr-worker and determines whether the document is an RG, CNH, income proof, address proof, or one of several other categories that CredityFlow needs for its lending workflow.

Classification uses a two-tier approach: fast rule-based heuristics handle the common, high-confidence cases, and an ML model catches the rest. This keeps latency low for the 85%+ of documents that are straightforward while still handling edge cases.

Built in **Python/FastAPI**, owned by **team-documents**, part of the **Document Management** domain.

## Getting Started

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

docker-compose up -d postgres   # For classification_db

uvicorn classifier.main:app --reload --port 8081
```

The ML model weights are downloaded from S3 on startup. For local development, a smaller test model is bundled in `models/test/`.

To run the worker mode (consuming from SQS):
```bash
python -m classifier.worker --profile local
```

### Tests
```bash
pytest tests/ -v
pytest tests/integration/ --run-integration
```

## Architecture

Dual-mode service: it runs as both an SQS consumer (primary path) and exposes a REST endpoint for manual reclassification.

The classification pipeline:
1. Parse the OCR result payload (extracted text + structured fields)
2. Run rule-based classifier: pattern matching on known keywords, field structures, and document layout markers
3. If rule-based confidence >= 0.85, use that result
4. Otherwise, invoke the ML model (a fine-tuned text classifier)
5. Persist the result and publish `documents_classified`

The ML model is versioned and loaded at startup. Switching models requires a deployment. Classification results include the model version for traceability.

Refer to [architecture.md](architecture.md) for integration patterns and ADR notes.

## API Reference

### Classify Document (Internal)
```
POST /api/v1/classify
Content-Type: application/json

{
  "document_id": "doc-456",
  "extracted_text": "REPUBLICA FEDERATIVA DO BRASIL ...",
  "extracted_fields": { ... },
  "override_category": null
}
```

The `override_category` field is used by operators to manually correct a classification. When set, the service persists the override and logs it for retraining purposes.

## Configuration

| Variable | Purpose |
|----------|---------|
| `DATABASE_URL` | PostgreSQL connection for document_classification_db |
| `SQS_QUEUE_URL` | Queue for ocr_completed events |
| `SNS_TOPIC_ARN` | Topic for documents_classified events |
| `MODEL_PATH` | Local or S3 path to ML model weights |
| `RULE_CONFIDENCE_THRESHOLD` | Minimum confidence to accept rule-based result (default: 0.85) |

## Deployment

EKS deployment. The ML model adds about 500MB to the container image.

- **CPU**: 500m / 1500m (ML inference is CPU-bound)
- **Memory**: 1Gi / 2Gi
- **Replicas**: 2 (staging), 4 (production)

Model updates require a new Docker image build and deployment. Hot-swapping models at runtime is not yet supported.

## Monitoring

Tracked metrics:
- `classification_total` by category and method (rule vs. ML)
- `classification_confidence_score` histogram
- `ml_model_inference_duration_seconds`
- `manual_reclassification_total`

The "Classification Distribution" dashboard shows the breakdown of categories over time and highlights when the `OTHER` bucket grows, which usually signals a new document format that needs rule or model updates.

Alert: `OTHER` category exceeds 15% of classification volume.

## Known Issues

- **New document formats**: When Brazilian authorities update the layout of identity documents, both rules and the ML model can break. Response time to these changes is typically 1-2 weeks.
- **Multi-page documents**: Each page is classified independently. A bank statement spread across 3 pages may get different classifications per page. Grouping logic is not implemented yet.
- **Model staleness**: The ML model is retrained quarterly. Between retraining cycles, accuracy can drift as document populations shift.
