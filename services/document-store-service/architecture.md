# Document Store Service (svc-09)

**Domain:** Documents
**Stack:** Kotlin / Spring Boot
**Database:** MongoDB (`document_store`) + AWS S3 (`credityflow-documents-archive`)
**Deployment:** AWS EKS

## Component Overview

While the Document Upload Service handles ingestion, the Document Store Service is responsible for the long-term lifecycle of documents within CredityFlow. It provides versioned storage, retrieval, and access control for all documents associated with a proposal. This is the service that other domains call when they need to fetch a document or verify its existence.

The service maintains a clear separation between metadata (MongoDB) and binary content (S3):

- **Metadata Layer (MongoDB):** Each document record includes the document ID, proposal ID, customer ID, document type, classification result, OCR result reference, validation status, version history, and access audit log. The flexible schema accommodates type-specific metadata fields without requiring migrations.

- **Binary Layer (S3):** Document files are stored in S3 with server-side encryption (SSE-S3). Each version of a document gets a distinct S3 object key following the pattern `{proposal_id}/{document_id}/v{version}/{filename}`. S3 Object Lock is enabled in compliance mode for regulatory retention requirements.

- **Version Manager:** When a customer re-uploads a document (e.g., after a validation failure), the service creates a new version rather than overwriting. The previous version remains accessible for audit purposes. The latest version is always the default when retrieving.

- **Access Control Gateway:** All document access goes through this service, which checks the caller's permissions before generating a time-limited S3 presigned GET URL. Access events are logged to the audit trail in MongoDB.

## Data Flow

Documents arrive at the store through two paths:

**Primary path:** After a document passes through upload, OCR, classification, and validation, the Document Store Service receives a `DocumentValidated` event and marks the document as "complete" in the metadata store. The binary was already in S3 from the upload step; the store service simply records the finalized metadata.

**Re-upload path:** When a document fails validation, the customer submits a replacement. The Upload Service sends a new `DocumentUploaded` event with the same `document_id` but incremented version. The Store Service creates a new version entry while preserving the old one.

**Retrieval:** Other services (Credit, Fraud, Appraisal) request documents via REST. The Store Service verifies permissions, generates a presigned GET URL, and returns it. The caller downloads directly from S3.

## Integration Patterns

- **Inbound (async):** Consumes `DocumentValidated`, `DocumentUploaded`, and `DocumentClassified` events from SNS/SQS to maintain the metadata lifecycle.
- **Inbound (sync):** REST API for document retrieval, listing, and version queries. Consumed by `income-verification-service`, `appraisal-orchestrator-service`, and backoffice.
- **Outbound:** Presigned S3 URLs for binary retrieval. No outbound events -- this is a terminal service in the document pipeline.

## ADR-001: Separate Store Service from Upload Service

**Context:** Early in the project, upload and storage were a single service. As the codebase grew, the responsibilities diverged: uploads need high throughput and virus scanning, while storage needs versioning, access control, and audit logging. The combined service was becoming difficult to maintain and had conflicting scaling characteristics.

**Decision:** Split into two services. The Upload Service (`svc-05`) handles ingestion; the Store Service (`svc-09`) handles lifecycle management and retrieval.

**Consequences:**
- Each service can be scaled independently. Upload spikes during onboarding campaigns don't affect document retrieval performance.
- Slight increase in operational overhead (two deployments, two monitoring dashboards).
- Clean separation of concerns makes it easier for different team members to own each service.

## ADR-002: S3 Object Lock for Regulatory Retention

**Context:** Brazilian financial regulations require that documents supporting a lending decision be retained for a minimum of 5 years. Accidental or malicious deletion of documents would be a compliance violation.

**Decision:** Enable S3 Object Lock in compliance mode with a 5-year retention period on the archive bucket. Objects cannot be deleted or overwritten until the retention period expires, not even by the root AWS account.

**Consequences:**
- Guaranteed regulatory compliance for document retention.
- Storage costs increase over time as documents accumulate without deletion. Mitigated by S3 Intelligent Tiering, which automatically moves infrequently accessed documents to cheaper storage classes.
- Correcting a wrongly uploaded document requires creating a new version rather than replacing the old one, which aligns with the versioning model.
