# OCR Worker (svc-06)

**Domain:** Documents
**Stack:** Python / FastAPI (worker mode)
**Database:** None (stateless)
**Deployment:** AWS EKS (GPU-enabled node group)

## Component Overview

The OCR Worker is a stateless, queue-driven processing unit. It has no database, no REST API exposed to other services, and no persistent state. Its sole job is to pull document images from an SQS queue, extract text using a combination of Tesseract OCR and custom ML models, and publish the results as events.

Think of it as a pure function deployed as a service: image bytes go in, structured text comes out.

The worker consists of three internal stages, executed as a pipeline for each message:

1. **Image Preprocessor** -- downloads the document image from S3 (using the object key from the SQS message), applies deskewing, noise reduction, and contrast normalization. These preprocessing steps are critical for Brazilian identity documents, which are often photographed under poor lighting conditions.

2. **OCR Engine** -- runs Tesseract 5 with Portuguese language packs as the first pass. For fields that Tesseract handles poorly (handwritten signatures, stamps, faded text), a custom CNN-based model trained on CredityFlow's labeled dataset takes over. The custom model specializes in extracting CPF numbers, dates, and monetary values from Brazilian financial documents.

3. **Result Publisher** -- packages the extracted text into a structured payload (field names mapped to extracted values with confidence scores) and publishes a `DocumentOCRCompleted` event to SNS. If OCR confidence falls below 60% on any critical field, a `DocumentOCRFailed` event is published instead, routing the document to manual review.

## Data Flow

```
SQS (document-ocr-queue)
    |
    v
[Download image from S3]
    |
    v
[Preprocess: deskew, denoise, normalize]
    |
    v
[Tesseract OCR pass]
    |
    v
[Custom ML model pass for low-confidence fields]
    |
    v
[Merge results, compute confidence scores]
    |
    v
SNS (document-events): DocumentOCRCompleted / DocumentOCRFailed
```

The worker processes one message at a time per pod replica. Horizontal scaling is managed by KEDA (Kubernetes Event Driven Autoscaling), which monitors the SQS queue depth and spins up additional pods when the backlog exceeds a threshold. Under normal load, 2 replicas suffice. During onboarding campaigns, the cluster can scale to 10+ replicas.

## Integration Patterns

- **Inbound:** SQS queue (`document-ocr-queue`) subscribed to the `DocumentUploaded` events from Document Upload Service. Message visibility timeout is set to 5 minutes to accommodate slow OCR processing.
- **Outbound:** SNS events only. No REST API, no database writes. The worker is invisible to the rest of the platform except through its events.
- **S3:** Read-only access to the documents bucket for fetching source images.

This is a deliberately isolated component. It can be redeployed, scaled, or even rewritten in a different language without affecting any other service, as long as the SQS input contract and SNS output contract remain stable.

## ADR-001: Stateless Worker over Stateful Service

**Context:** An earlier design had the OCR capability embedded within the Document Upload Service as a synchronous step. This created long request timeouts (some documents take 30+ seconds to OCR), coupled the upload flow to OCR availability, and made scaling difficult since upload and OCR have very different resource profiles.

**Decision:** Extract OCR into a standalone stateless worker that processes from an SQS queue. No database, no API surface. Communication is entirely event-driven.

**Consequences:**
- Independent scaling: OCR workers run on GPU nodes; upload service runs on standard nodes.
- Upload latency dropped from 30-45 seconds to under 2 seconds (upload is now decoupled from processing).
- Increased end-to-end latency for the overall onboarding flow, since OCR is now asynchronous. Mitigated by frontend UX that shows a processing indicator.

## ADR-002: Hybrid Tesseract + Custom ML Approach

**Context:** Pure Tesseract OCR achieved only 72% accuracy on Brazilian identity documents in production. A fully custom ML model was considered but would require 6+ months of development and a large labeled dataset.

**Decision:** Use Tesseract as the primary engine and layer a custom CNN model on top for specific field types (CPF, dates, monetary values) where Tesseract underperforms. The custom model was trained on 15,000 labeled document images from CredityFlow's existing portfolio.

**Consequences:**
- Overall accuracy improved to 91% on critical fields.
- Dual-engine approach increases per-document processing time by ~40%, acceptable given the async nature of the worker.
- The custom model requires periodic retraining as new document formats are encountered; this is managed via a monthly batch retraining pipeline.
