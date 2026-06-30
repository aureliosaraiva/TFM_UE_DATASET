# Appraisal Report Service

## Quick Reference

| Field | Value |
|-------|-------|
| Service ID | svc-26 |
| Domain | Appraisal |
| Team | team-appraisal |
| Stack | Kotlin / Spring Boot |
| Database | PostgreSQL (metadata) + S3 (PDF storage) |

## Overview

The Appraisal Report Service is the final step in the appraisal pipeline. It takes the raw valuation data produced by the vehicle or property appraisal services and transforms it into a standardised, downloadable PDF report. These reports serve multiple audiences: backoffice analysts reviewing loan files, compliance teams conducting audits, and external regulators requesting documentation.

Every report is persisted in S3 with a 10-year retention policy (per Brazilian Central Bank regulations) and indexed in PostgreSQL for fast retrieval. The service generates reports automatically upon receiving valuation events and signals the completion of the appraisal workflow by publishing `appraisal_completed`.

## Getting Started

```bash
# Dependencies
docker-compose up -d postgres localstack   # localstack simulates S3

# Configure
export DATABASE_URL=jdbc:postgresql://localhost:5432/appraisal_report_db
export DATABASE_USERNAME=report_svc
export DATABASE_PASSWORD=changeme
export KAFKA_BOOTSTRAP_SERVERS=localhost:9092
export AWS_S3_BUCKET=appraisal-reports
export AWS_S3_ENDPOINT=http://localhost:4566  # localstack
export AWS_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test

# Run
./gradlew bootRun

# Tests
./gradlew test
./gradlew integrationTest
```

Health check at `GET /actuator/health`. The service verifies S3 bucket access during startup.

## Architecture

The service operates as an event-driven report generator:

```
vehicle_appraised   --+
                      +--> [Report Generator] --> S3 Upload --> appraisal_completed
property_appraised  --+
```

Internally, the generation pipeline:

1. **Event ingestion** — Kafka consumer receives valuation event
2. **Data enrichment** — Fetches supplementary metadata (proposal details, customer info) if needed
3. **Template selection** — Picks the appropriate PDF template (vehicle vs. property)
4. **PDF rendering** — Uses a Thymeleaf + OpenPDF pipeline to generate the document
5. **S3 upload** — Uploads the PDF to the `appraisal-reports` bucket with server-side encryption
6. **Metadata persistence** — Saves `AppraisalReport` record in PostgreSQL
7. **Event publication** — Publishes `appraisal_completed` to signal workflow completion

PDF downloads are served via **pre-signed S3 URLs** with 15-minute expiry, so the service itself never streams large files through its own pods.

See [architecture.md](./architecture.md) for the full component diagram and ADRs.

## API Reference

### GET /api/v1/appraisal-reports/{id}

Returns report metadata and valuation summary.

```json
{
  "report_id": "uuid",
  "proposal_id": "uuid",
  "collateral_type": "VEHICLE",
  "appraised_value": 82450.00,
  "status": "GENERATED",
  "generated_at": "2026-03-29T12:00:00Z",
  "s3_key": "reports/2026/03/uuid.pdf"
}
```

### GET /api/v1/appraisal-reports/{id}/download

Returns a JSON object containing a pre-signed S3 URL for the PDF.

```json
{
  "download_url": "https://appraisal-reports.s3.amazonaws.com/reports/2026/03/uuid.pdf?X-Amz-...",
  "expires_at": "2026-03-29T12:15:00Z"
}
```

The URL expires after 15 minutes. Clients should not cache it beyond that window.

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | — |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka broker addresses | `localhost:9092` |
| `AWS_S3_BUCKET` | S3 bucket for reports | `appraisal-reports` |
| `AWS_S3_ENDPOINT` | S3 endpoint (override for localstack) | — |
| `AWS_REGION` | AWS region | `sa-east-1` |
| `PRESIGNED_URL_EXPIRY_MINUTES` | Pre-signed URL validity | `15` |
| `PDF_TEMPLATE_DIR` | Directory containing PDF templates | `classpath:templates/` |
| `KAFKA_CONSUMER_GROUP` | Consumer group ID | `appraisal-report-cg` |

## Deployment

```bash
helm upgrade --install appraisal-report ./charts/appraisal-report-service \
  --namespace appraisal \
  --values values-production.yaml
```

**Resources:** 1Gi memory / 500m CPU (request). PDF generation is CPU- and memory-intensive — do not under-provision. HPA scales on CPU utilisation (target 70%), 2-6 replicas.

**IAM requirements:** The EKS service account must have `s3:PutObject`, `s3:GetObject`, and `s3:GeneratePresignedUrl` permissions on the target bucket.

**S3 bucket configuration:** Server-side encryption (AES-256) enabled, versioning enabled, lifecycle rule to transition to Glacier after 1 year and retain for 10 years.

## Monitoring

- **Dashboard:** "Appraisal Reports Overview" — generation volume, download rate, generation latency
- **Dashboard:** "Storage Analytics" — S3 bucket growth, report size distribution

**Alerts:**
- `report_generation_failure` — errors > 2% over 10 min
- `s3_upload_failure` — any upload error (critical — blocks appraisal workflow)
- `report_generation_latency_high` — p95 > 30s

Logs include `report_id`, `proposal_id`, and `collateral_type` for filtering. PDF generation errors include template name and rendering stage for debugging.

## Known Issues

1. **Memory pressure during batch reprocessing** — If many valuation events arrive simultaneously (e.g., after an upstream service recovers from downtime), concurrent PDF generation can cause OOM kills. The Kafka consumer is configured with `max.poll.records=5` to throttle throughput, but this may not be sufficient in extreme scenarios. A dedicated job queue with backpressure is under consideration (APPR-501).

2. **Template changes require redeployment** — PDF templates are bundled in the application JAR. Changing formatting, adding fields, or updating branding requires a full build and deploy cycle. Externalising templates to S3 is planned (APPR-510).

3. **Pre-signed URL expiry confusion** — Some downstream consumers cache the download URL and attempt to use it after expiry. The API response includes `expires_at` explicitly, but not all consumers respect it. Documentation has been updated, but occasional support tickets persist.
