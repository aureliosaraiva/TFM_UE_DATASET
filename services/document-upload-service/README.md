# document-upload-service

## Overview

This service handles the ingestion of documents into CredityFlow's document processing pipeline. When customers apply for a loan, they need to submit identity documents (RG, CNH), income proofs, and collateral documentation. The document-upload-service accepts these files, stores them securely in S3, tracks metadata in MongoDB, and kicks off the downstream processing chain.

It is the first service in the document pipeline: upload -> OCR -> classification -> validation -> final storage.

**Team**: team-documents | **Domain**: Document Management

## Getting Started

You will need:
- JDK 17+
- Docker Compose (for MongoDB and LocalStack)
- LocalStack for S3 emulation locally

```bash
docker-compose up -d mongodb localstack
./gradlew bootRun --args='--spring.profiles.active=local'
```

LocalStack provides an S3-compatible endpoint at `http://localhost:4566`. The local profile auto-creates the required bucket on startup.

```bash
# Upload a test document
curl -X POST http://localhost:8080/api/v1/documents/upload \
  -F "file=@test-doc.pdf" \
  -F "proposal_id=prop-123" \
  -F "document_type=CNH"
```

## Architecture

Spring Boot application with MongoDB for metadata and S3 for file storage. Uses Spring WebFlux for non-blocking file upload handling, which is important given that large file uploads can tie up threads.

The service creates an **UploadSession** when it receives a `proposal_started` event. This session defines which documents are required for the proposal's product type. As documents are uploaded, the session tracks progress. Once all required documents are in, downstream processing begins.

See [architecture.md](architecture.md) for component diagrams and integration details.

## API Reference

### Upload Document
```
POST /api/v1/documents/upload
Content-Type: multipart/form-data

Fields:
  file          - The document file (PDF, JPEG, or PNG, max 10MB)
  proposal_id   - Associated proposal ID
  document_type - One of: RG, CNH, CPF_CARD, INCOME_PROOF, ADDRESS_PROOF,
                  VEHICLE_REGISTRATION, PROPERTY_DEED
```

Returns: `201 Created` with document ID and S3 reference.

### Get Document Metadata
```
GET /api/v1/documents/{id}
```
Returns metadata and a pre-signed S3 URL valid for 15 minutes.

### Get Upload Session
```
GET /api/v1/sessions/{sessionId}
```
Returns session status with a checklist of required vs. received documents.

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `MONGODB_URI` | MongoDB connection string | `mongodb://localhost:27017/document_meta_db` |
| `AWS_S3_BUCKET` | S3 bucket for document files | `credityflow-documents-upload` |
| `AWS_S3_ENDPOINT` | S3 endpoint (for LocalStack) | _(uses default AWS)_ |
| `AWS_SNS_TOPIC_ARN` | SNS topic for document events | - |
| `MAX_FILE_SIZE` | Maximum upload file size | `10MB` |

## Deployment

Deployed on EKS via Helm. The S3 bucket is provisioned by Terraform with:
- SSE-S3 encryption enabled
- Versioning enabled
- Lifecycle rule: transition to Glacier after 7 years
- Bucket policy restricts access to the service's IAM role

**Resources**: 500m/1000m CPU, 512Mi/1024Mi memory, 2-3 replicas.

## Monitoring

- **document_upload_total**: volume by document type
- **upload_duration_seconds**: end-to-end upload latency
- **s3_put_errors_total**: S3 write failures

Alerts fire on S3 failure rate > 1%, upload latency p99 > 10s, or MongoDB write errors.

The "Document Upload Overview" Grafana dashboard shows upload volume trends, file size distribution, and error rates.

## Known Issues

- **No resumable uploads**: Large files on slow mobile connections can fail. Implementing tus.io protocol is on the backlog.
- **No virus scanning**: Uploaded files are not scanned for malware. A ClamAV integration is planned.
- **Session cleanup**: Abandoned upload sessions (started but never completed) are not automatically cleaned up. A scheduled job is needed.
