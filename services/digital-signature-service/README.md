# digital-signature-service

> Orchestrates the digital signing of loan contracts through DocuSign-BR, turning compliance-approved contracts into legally binding documents.

**Team:** team-contracts | **Domain:** Contract | **ID:** svc-29

---

## Overview

When a contract passes compliance checks, this service takes over. It packages the contract into a DocuSign-BR envelope, sends it to the required signers, and waits for callbacks confirming the signature. Once all parties have signed, it retrieves the digital certificate and notifies downstream services that the contract is ready for storage and registry.

This service handles PII (names, CPF numbers, emails) and applies column-level encryption on all sensitive fields stored in PostgreSQL.

## Getting Started

**Requirements:** JDK 17+, Docker, Kafka, access to DocuSign-BR sandbox credentials.

```bash
# Spin up local infra (Postgres + Kafka)
docker compose up -d

# Export DocuSign sandbox credentials
export DOCUSIGN_API_KEY=your-sandbox-key
export DOCUSIGN_ACCOUNT_ID=your-sandbox-account

# Start the service
./gradlew bootRun --args='--spring.profiles.active=local'
```

Verify it is running: `curl http://localhost:8081/actuator/health`

### Tests

```bash
./gradlew test                    # Unit tests (DocuSign client is mocked)
./gradlew integrationTest         # Needs Docker; uses WireMock for DocuSign
```

## Architecture

Hexagonal architecture with the DocuSign-BR client isolated behind a port interface, making it straightforward to swap providers or mock in tests.

Core flow: Kafka event in (`compliance_checked`) --> DocuSign-BR API call --> Kafka events out (`signature_requested`, `contract_signed`).

The callback endpoint is exposed publicly (behind API Gateway with IP allowlisting for DocuSign-BR) and validates every incoming request using HMAC signatures.

See [architecture.md](./architecture.md) for component diagrams and ADRs.

## API Reference

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/signatures/request` | Create a signature request for a contract |
| GET | `/api/v1/signatures/{id}/status` | Check current signature status |
| POST | `/api/v1/signatures/callback` | DocuSign-BR webhook receiver |

### POST /api/v1/signatures/request

```json
{
  "contract_id": "uuid",
  "signers": [
    { "name": "João Silva", "cpf": "123.456.789-00", "email": "joao@email.com" }
  ],
  "document_url": "s3://contracts/draft/uuid.pdf"
}
```

Returns `201 Created` with the signature request ID and DocuSign envelope ID.

### POST /api/v1/signatures/callback

Called by DocuSign-BR. The service validates the `X-DocuSign-Signature` header before processing. Returns `200 OK` on success, `401 Unauthorized` on invalid signature.

## Configuration

| Variable | Description | Default |
|---|---|---|
| `SPRING_DATASOURCE_URL` | JDBC connection to signature_db | `jdbc:postgresql://localhost:5432/signature_db` |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka brokers | `localhost:9092` |
| `DOCUSIGN_API_KEY` | DocuSign-BR API key | — (required) |
| `DOCUSIGN_ACCOUNT_ID` | DocuSign-BR account | — (required) |
| `DOCUSIGN_BASE_URL` | DocuSign-BR endpoint | `https://api.docusign-br.com` |
| `DOCUSIGN_CALLBACK_URL` | Public URL for webhooks | — (required in prod) |
| `ENCRYPTION_KEY` | AES key for PII column encryption | — (required) |

## Deployment

Runs on EKS. Production uses 3 replicas minimum for high availability. The callback endpoint is routed through the API Gateway with DocuSign-BR source IPs allowlisted.

Secrets (`DOCUSIGN_API_KEY`, `ENCRYPTION_KEY`) are injected from AWS Secrets Manager via External Secrets Operator.

## Monitoring

- **Grafana dashboard:** "Signature Pipeline" — request volume, time-to-sign distribution, completion rates.
- **Grafana dashboard:** "DocuSign-BR Health" — API call latency, error rates, circuit breaker state.
- **PagerDuty alert:** DocuSign-BR API errors exceeding 10/min for 5 minutes triggers P1.
- **PagerDuty alert:** Signatures pending longer than 72 hours triggers P3 for follow-up.

## Known Issues

1. **Single provider dependency.** DocuSign-BR is the only supported signature provider. If their service is degraded, there is no fallback. A multi-provider abstraction is on the roadmap.
2. **Callback retries.** DocuSign-BR retries callbacks up to 3 times. If all retries fail (e.g., during a deployment), the signature event is lost until the next polling cycle (every 30 minutes).
3. **PII in logs.** Structured logging masks PII fields, but stack traces from deserialization errors may leak signer data. Log scrubbing is applied at the Fluentd layer as a safety net.
