# Architecture — contract-store-service

## Component Diagram

```
                          ┌─────────────────────────────────┐
  contract_signed         │    contract-store-service        │
  ──────────────────►     │                                  │
  (Kafka)                 │  ┌────────────────────────────┐  │
                          │  │   Kafka Consumer            │  │
                          │  └────────────┬───────────────┘  │
                          │               │                   │
                          │               ▼                   │
                          │  ┌────────────────────────────┐  │
                          │  │   Contract Storage Service  │  │
                          │  │   (Domain Core)             │  │
                          │  └──────┬──────────────┬──────┘  │
                          │         │              │          │
                          │         ▼              ▼          │
                          │  ┌────────────┐ ┌────────────┐   │
                          │  │  S3 Adapter│ │  MongoDB    │   │
                          │  │            │ │  Adapter    │   │
                          │  └─────┬──────┘ └──────┬─────┘   │
                          │        │               │          │
                          │  ┌─────────────────────────────┐ │
  REST Clients ──────►    │  │   REST Controller            │ │
  GET /contracts/...      │  │   (download + versions)      │ │
                          │  └─────────────────────────────┘ │
                          └────────┼───────────────┼─────────┘
                                   │               │
                                   ▼               ▼
                            ┌──────────┐    ┌──────────────┐
                            │  AWS S3  │    │   MongoDB    │
                            │ (PDFs)   │    │ (metadata)   │
                            └──────────┘    └──────────────┘
```

## Data Flow

1. **Event Ingestion.** The Kafka consumer receives `contract_signed` events. Each event carries a contract ID, a temporary URL to the signed PDF, the signature certificate reference, and metadata (product type, signing timestamp, signers).

2. **Document Storage.** The Storage Service downloads the signed PDF from the temporary URL, computes a SHA-256 checksum, and uploads it to S3 under the key pattern `contracts/{contract_id}/v{version}.pdf`. Server-side encryption is applied automatically by the bucket policy.

3. **Metadata Persistence.** A `StoredContract` document is upserted in MongoDB. If the contract already exists (e.g., an amendment or re-signing), a new `ContractVersion` subdocument is appended to the versions array. The MongoDB document includes the contract ID, S3 keys for all versions, checksums, timestamps, and product-specific metadata.

4. **Download Serving.** When a client requests a contract download, the REST controller looks up the MongoDB metadata, identifies the correct S3 key (latest or specified version), generates a pre-signed URL with a 15-minute TTL, and returns it. The client then downloads directly from S3 — the service never proxies the file content.

5. **Version Listing.** The versions endpoint reads the MongoDB document and returns the versions array with checksums and timestamps. This is used by backoffice for audit trail review and by customer service for dispute resolution.

## Integrations

| System | Direction | Protocol | Purpose |
|---|---|---|---|
| digital-signature-service | Inbound | Kafka | `contract_signed` events trigger archival |
| AWS S3 | Outbound | AWS SDK | PDF binary storage and pre-signed URL generation |
| MongoDB Atlas | Internal | MongoDB driver | Metadata and version history storage |

No outbound events are published. This service is a terminal node in the contract pipeline.

## ADR-001: MongoDB for Metadata Instead of PostgreSQL

**Status:** Accepted

**Context:** Contract metadata varies significantly by product type (vehicle loans include RENAVAM data, property loans include matricula data, personal loans have different fields). The team was spending excessive time on PostgreSQL schema migrations as new product types were added.

**Decision:** Use MongoDB for contract metadata storage. Each contract is a single document with a flexible schema. Common fields (contract_id, versions, checksums) are enforced at the application level via Kotlin data classes, while product-specific metadata is stored in a dynamic `metadata` map.

**Consequences:** Gained flexibility for product-specific fields without migrations. Lost referential integrity guarantees and SQL query capabilities. Reporting queries are more complex and run through MongoDB aggregation pipelines. The team must be disciplined about application-level validation to prevent data quality issues.

## ADR-002: Pre-Signed URLs Instead of Proxying Downloads

**Status:** Accepted

**Context:** The service needs to serve contract PDFs to internal consumers. Two approaches were evaluated: (a) proxy the S3 content through the service, or (b) generate pre-signed S3 URLs and redirect the client.

**Decision:** Use pre-signed URLs with a 15-minute TTL. The service generates the URL and returns it to the client, which then downloads directly from S3.

**Consequences:** Eliminates the service as a bandwidth bottleneck — S3 handles all download traffic directly. Reduces memory pressure on service pods (no large file buffering). The tradeoff is that clients must handle the redirect, and URLs are time-limited. For Glacier-archived documents, the service must first initiate a restore and return a 202 with a retry-after header, adding complexity to the client integration.
