# contract-store-service

The final stop in the contract lifecycle. Once a loan contract is digitally signed, this service archives the document in S3 and indexes its metadata in MongoDB, providing a versioned, auditable repository that meets BACEN's 5-year retention requirement.

| Field | Value |
|-------|-------|
| Service ID | svc-30 |
| Domain | Contract |
| Owner | team-contracts |
| Data Classification | PII + Financial |

## Getting Started

You will need JDK 17+, Docker (for MongoDB), and either LocalStack or an AWS sandbox account for S3.

```bash
# Start MongoDB via Docker
docker compose up -d mongo

# Configure S3 (LocalStack for local dev)
export AWS_ENDPOINT=http://localhost:4566
export S3_BUCKET_NAME=contracts-local
aws --endpoint-url=$AWS_ENDPOINT s3 mb s3://$S3_BUCKET_NAME

# Run
./gradlew bootRun --args='--spring.profiles.active=local'
```

Health endpoint: `GET /actuator/health` on port 8082.

### Testing

Unit tests mock both MongoDB and S3. Integration tests use Testcontainers for MongoDB and LocalStack for S3.

```bash
./gradlew test
./gradlew integrationTest
```

## Architecture

The service is intentionally simple: it consumes a single Kafka event, stores a file, and writes metadata. There is no complex business logic.

```
contract_signed (Kafka) --> [Consumer] --> [Storage Service] --> S3 + MongoDB
                                                  |
                                          [REST Controller] <-- GET requests
```

MongoDB was chosen over PostgreSQL here because contract metadata varies by product type and evolves frequently. The schemaless model avoids constant migrations.

Detailed diagrams and ADRs are in [architecture.md](./architecture.md).

## API Reference

### GET /api/v1/contracts/{id}/download

Returns a pre-signed S3 URL (valid for 15 minutes) for the latest version of the signed contract. Add `?version=N` to download a specific historical version.

**Response (200):**
```json
{
  "contract_id": "uuid",
  "version": 1,
  "download_url": "https://s3.amazonaws.com/...",
  "expires_at": "2026-03-29T10:15:00Z"
}
```

### GET /api/v1/contracts/{id}/versions

Lists all versions of a stored contract.

**Response (200):**
```json
{
  "contract_id": "uuid",
  "versions": [
    {
      "version": 1,
      "checksum_sha256": "abc123...",
      "stored_at": "2026-03-28T14:00:00Z",
      "size_bytes": 204800
    }
  ]
}
```

## Configuration

| Variable | Description | Default |
|---|---|---|
| `MONGO_URI` | MongoDB connection string | `mongodb://localhost:27017/contract_files` |
| `S3_BUCKET_NAME` | Bucket for contract PDFs | `contracts-prod` |
| `S3_REGION` | AWS region | `sa-east-1` |
| `AWS_ENDPOINT` | Custom S3 endpoint (for LocalStack) | — |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka brokers | `localhost:9092` |
| `PRESIGNED_URL_TTL_MINUTES` | Pre-signed URL expiry | `15` |

## Deployment

Runs on EKS with 2 replicas in production. S3 access is granted via IAM role for service accounts (IRSA). MongoDB runs as an Atlas managed cluster in the `sa-east-1` region.

S3 bucket configuration:
- Server-side encryption: SSE-S3 (AES-256)
- Versioning: enabled at the bucket level as a safety net
- Lifecycle: transition to Glacier after 730 days, delete after 2,555 days (7 years)

## Monitoring

Prometheus metrics at `/actuator/prometheus`. Key things to watch:

- **contracts_stored_total** should track closely with `contract_signed` event rate. A divergence indicates consumer lag or storage failures.
- **s3_upload_duration_seconds** p99 should stay under 2s. Spikes indicate S3 throttling or network issues.
- S3 upload failure alert triggers PagerDuty P1 if error rate exceeds 5% for 10 minutes.

## Known Issues

- **Glacier retrieval latency.** Contracts older than 2 years are in S3 Glacier. Downloading them requires a restore request that takes 3-5 hours. The API returns a 202 Accepted with a retry-after header in this case, but callers often do not handle it gracefully.
- **Duplicate versioning.** Kafka at-least-once delivery can result in duplicate `contract_signed` events creating redundant versions. A deduplication mechanism using the event ID is planned but not yet implemented.
- **No search capability.** There is no full-text or metadata search API. Finding contracts requires knowing the contract ID. A search feature backed by Elasticsearch is on the backlog.
