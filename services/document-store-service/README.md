# document-store-service

## Overview

The document-store-service is the final destination for validated documents in CredityFlow's document pipeline. Once a document passes all validation rules, this service copies it from the upload bucket to a permanent storage bucket, records its metadata, and manages its lifecycle for the duration of the loan (and beyond, per regulatory retention requirements).

It also serves as the primary read API for downstream consumers -- underwriters, compliance teams, and other services -- that need to access validated documents associated with a proposal.

**Stack**: Kotlin/Spring Boot, MongoDB + S3 | **Team**: team-documents | **Domain**: Document Management

## Getting Started

```bash
docker-compose up -d mongodb localstack
./gradlew bootRun --args='--spring.profiles.active=local'
```

LocalStack provides both the upload and final-storage S3 buckets locally. The service distinguishes between the two via configuration.

```bash
./gradlew test
./gradlew integrationTest    # Uses Testcontainers for MongoDB
```

### Quick Test

After starting the service, you can query for documents:
```bash
curl http://localhost:8080/api/v1/proposals/prop-123/documents/validated
```

## Architecture

Event-driven on the write path, REST-only on the read path. The service consumes `documents_validated` events, copies files between S3 buckets, creates metadata records, and publishes `documents_submitted` when a proposal's document set is complete.

The decision to use a separate S3 bucket for final storage (rather than reusing the upload bucket) was deliberate: the final bucket has stricter IAM policies, longer retention, and is the bucket referenced by compliance and audit processes.

MongoDB was chosen over PostgreSQL here because document metadata varies significantly by document type, and the flexible schema avoids constant migrations. The service shares the same MongoDB cluster as document-upload-service but uses separate collections prefixed with `store_`.

See [architecture.md](architecture.md) for data flow and the ADR on the dual-bucket strategy.

## API Reference

### Get Document
```
GET /api/v1/documents/{id}
```
Returns document metadata and a pre-signed S3 download URL (15-minute expiry).

### Get Validated Documents for Proposal
```
GET /api/v1/proposals/{id}/documents/validated
```
Returns a list of all validated documents for the specified proposal, each with metadata and a download URL. Supports pagination via `page` and `size` query parameters.

Both endpoints require service-to-service authentication. Pre-signed URLs are scoped to the requesting service's IAM context.

## Configuration

| Variable | Description |
|----------|-------------|
| `MONGODB_URI` | MongoDB connection string |
| `S3_UPLOAD_BUCKET` | Source bucket (document-upload-service's bucket) |
| `S3_STORE_BUCKET` | Destination bucket for final storage |
| `SQS_QUEUE_URL` | Queue for documents_validated events |
| `SNS_TOPIC_ARN` | Topic for documents_submitted events |
| `PRESIGNED_URL_EXPIRY_MINUTES` | Pre-signed URL validity (default: 15) |

## Deployment

EKS via Helm. The final storage S3 bucket is provisioned by Terraform with:
- SSE-KMS encryption (stricter than the upload bucket's SSE-S3)
- Object Lock in compliance mode for regulatory retention
- 10-year lifecycle policy before transition to Glacier Deep Archive
- Cross-region replication to a backup bucket

**Resources**: 500m/1000m CPU, 512Mi/1024Mi memory, 2 replicas.

## Monitoring

- `documents_stored_total` by category
- `document_retrieval_total` by caller service
- `s3_copy_duration_seconds`
- `proposal_document_completeness_ratio` -- tracks how many proposals have all required documents stored

Alert on S3 copy failures > 1%, retrieval latency p99 > 5s, or proposals stuck with incomplete documents for over 72 hours.

Dashboard: "Document Store Overview" in Grafana.

## Known Issues

- **S3 cross-bucket copy latency**: Copying large files between buckets adds 1-5 seconds of processing time per document. This is acceptable but adds up for proposals with many documents.
- **Shared MongoDB**: The MongoDB cluster is shared with document-upload-service. Under high upload volume, write contention on the cluster can affect read performance for this service. A read-preference configuration (`secondaryPreferred`) mitigates this partially.
- **No search**: There is no full-text search on document content. If an underwriter needs to find a specific document by content, they must browse by proposal. An Elasticsearch integration has been discussed but not prioritized.
- **Completeness logic**: The logic for determining whether all required documents are present is duplicated between this service and document-upload-service. This should be extracted into a shared library or a dedicated orchestration step.
