# Document Upload Service (svc-05)

**Domain:** Documents
**Stack:** Kotlin / Spring Boot
**Database:** MongoDB (`document_metadata`) + AWS S3 (`credityflow-documents`)
**Deployment:** AWS EKS

## Component Overview

This service handles the ingestion side of the document lifecycle. When a customer needs to submit identity documents, proof of income, or collateral documentation, the Document Upload Service is the entry point. It manages multipart uploads, performs virus scanning, generates S3 presigned URLs for direct browser-to-S3 uploads, and stores metadata about each uploaded file.

Key internal components:

- **Upload Controller** -- handles two upload flows: (a) server-side multipart upload for smaller files, and (b) presigned URL generation for larger files where the browser uploads directly to S3, bypassing the service for the payload transfer.
- **Virus Scanner** -- every uploaded file is routed through ClamAV (deployed as a sidecar container) before being accepted. Files that fail scanning are quarantined in a separate S3 prefix and flagged in metadata.
- **Metadata Store** -- MongoDB stores document metadata: file name, MIME type, upload timestamp, customer ID, proposal ID, scan status, and S3 object key. MongoDB was chosen over PostgreSQL here because document metadata schemas vary by document type and evolve frequently.
- **Event Emitter** -- upon successful upload and scan, a `DocumentUploaded` event is published to the `document-events` SNS topic, triggering downstream OCR and classification workflows.

## Data Flow

The typical upload flow proceeds as follows:

1. Frontend requests a presigned URL from the service, providing the target document type and proposal context.
2. Service generates an S3 presigned PUT URL (expires in 15 minutes) and returns it along with a `document_id`.
3. Frontend uploads the file directly to S3 using the presigned URL.
4. S3 triggers a Lambda notification that calls back to the service, confirming the upload.
5. Service pulls the file from S3, pipes it through ClamAV, and updates the metadata record with the scan result.
6. If clean, a `DocumentUploaded` event is published. If infected, a `DocumentQuarantined` event is published and the file is moved to the quarantine prefix.

For the server-side multipart flow (used by the backoffice), the file goes through the service directly, following steps 5-6 after persistence.

## Integration Patterns

- **S3 Presigned URLs:** Offloads bandwidth from the service pods. The service acts as a control plane, not a data plane, for large uploads.
- **ClamAV Sidecar:** Runs as a separate container in the same pod. Communication happens over a Unix socket for minimal latency.
- **SNS/SQS:** `DocumentUploaded` events fan out to `ocr-worker`, `document-classifier-service`, and `proposal-service`.
- **Synchronous REST:** The backoffice and frontend query document status through GET endpoints.

## ADR-001: Presigned URLs for Direct S3 Upload

**Context:** Initially, all document uploads were proxied through the service. During load testing, the service became a bottleneck at 200+ concurrent uploads, consuming excessive memory for buffering large files (some collateral photos exceed 10 MB).

**Decision:** Implement S3 presigned URLs for frontend uploads. The service generates a time-limited, scoped PUT URL; the browser uploads directly to S3. The service is notified via S3 event notification after the upload completes.

**Consequences:**
- Dramatically reduced memory and CPU usage on service pods; upload throughput is now bounded by S3, not the service.
- Added complexity: the service must handle the callback flow and deal with cases where the presigned URL is generated but the upload never completes (cleanup via S3 lifecycle rules after 24 hours).
- CORS configuration on the S3 bucket must be carefully maintained.

## ADR-002: MongoDB for Document Metadata

**Context:** The Documents domain needed a metadata store. PostgreSQL was the platform default, but document metadata varies significantly by type -- an identity document has different fields than a property deed. Schema migrations for every new document type would slow the team.

**Decision:** Use MongoDB for document metadata storage. Each document type can have its own metadata shape within a single `documents` collection, leveraging MongoDB's flexible schema.

**Consequences:**
- Faster iteration when onboarding new document types; no migration needed.
- Lose transactional guarantees with other PostgreSQL-based services; acceptable since document metadata is self-contained.
- The team adopted JSON Schema validation in MongoDB to prevent completely unstructured data from being written.
