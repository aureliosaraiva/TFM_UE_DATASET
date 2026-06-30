# Architecture: Appraisal Report Service (svc-26)

## At a Glance

| Attribute | Value |
|-----------|-------|
| Domain | Appraisal |
| Language | Kotlin |
| Framework | Spring Boot 3.x |
| Storage | PostgreSQL (report metadata), S3 (generated PDFs) |
| Inputs | SNS/SQS events from vehicle and property appraisal services |
| Outputs | Signed S3 URLs for PDF download |

## What This Service Does

The Appraisal Report Service is a document generation pipeline. It consumes valuation events from the appraisal domain, assembles structured data into a branded PDF report, and stores the output in S3. It does not perform valuations itself -- it is a pure consumer of appraisal results.

Reports are consumed by loan officers reviewing collateral, by customers via the customer portal, and by external auditors during compliance reviews.

## Components

**SQS Listener (`AppraisalEventConsumer`):** Subscribes to `appraisal-events-report-queue`. Deserializes `VehicleAppraised` and `PropertyAppraised` events and dispatches them to the report generation pipeline.

**Report Assembler (`ReportDataAssembler`):** Enriches the event payload by fetching supplementary data -- customer name from the customer service, collateral photos from S3, and any prior appraisal history from the appraisal services. Uses parallel async calls with a 5-second timeout per dependency.

**PDF Renderer (`PdfRenderingService`):** Uses Apache PDFBox with Thymeleaf-based HTML templates. Templates are versioned and stored in the classpath. The rendered PDF includes the CredityFlow watermark, QR code linking to the digital verification page, and a tamper-evident hash in the footer.

**S3 Storage (`ReportStorageService`):** Uploads the PDF to `s3://credityflow-appraisal-reports/{year}/{month}/{appraisalId}.pdf`. Generates pre-signed URLs (validity: 1 hour) for secure access.

**Report Metadata Repository:** Stores generation status, S3 key, version, and access log in PostgreSQL.

## Data Flow

```
SNS (appraisal-events)
        |
        v
SQS (appraisal-events-report-queue)
        |
        v
AppraisalEventConsumer
        |
        v
ReportDataAssembler  ----> [customer-service, appraisal services] (REST)
        |
        v
PdfRenderingService  ----> Thymeleaf template + PDFBox
        |
        v
ReportStorageService ----> S3 (credityflow-appraisal-reports)
        |
        v
PostgreSQL (report metadata, status: GENERATED)
        |
        v
SNS (report-events) --> ReportGenerated event
```

## Integration Patterns

This service is **event-driven on the input side** and **REST on the output side**:
- It subscribes to appraisal events via SQS (fan-out from the shared appraisal SNS topic).
- It exposes a REST API (`GET /v1/reports/{appraisalId}`) that returns report metadata and a pre-signed download URL.
- It makes synchronous REST calls to enrich report data but treats all enrichment as optional -- if a dependency is unavailable, the report is generated with placeholder text and flagged for regeneration.

**Retry semantics:** SQS visibility timeout is set to 120 seconds (PDF generation is CPU-intensive). Failed messages are retried 3 times before moving to a DLQ. A scheduled job processes the DLQ daily.

## ADR

### ADR-026-001: Server-side PDF generation over client-side rendering

**Context:** An alternative design considered was returning structured JSON to the frontend and generating the PDF in the browser. This would offload compute from the backend.

**Decision:** Generate PDFs server-side using PDFBox. This ensures consistent formatting across all access channels (web, mobile, email attachment) and allows embedding tamper-evident hashes that a client-side approach could not securely produce.

**Consequences:** Higher CPU usage on the service (mitigated by horizontal scaling on EKS). PDF templates are versioned in source control, not in a CMS. Template changes require a deployment.

### ADR-026-002: Pre-signed S3 URLs instead of proxying binary content

**Context:** Early design had the BFF proxying PDF downloads through the report service. This created unnecessary load and memory pressure.

**Decision:** Return pre-signed S3 URLs with 1-hour expiry. The client downloads directly from S3.

**Consequences:** Eliminates the service as a bottleneck for large file downloads. Requires the client to handle URL expiry gracefully (re-request if expired). S3 access logs provide an additional audit trail of who downloaded reports.
