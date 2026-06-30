# Architecture — digital-signature-service

## Component Diagram

```
                            ┌────────────────────────────────┐
  compliance_checked        │   digital-signature-service    │
  ────────────────────►     │                                │
  (Kafka)                   │  ┌──────────────────────────┐  │
                            │  │  Kafka Consumer          │  │
                            │  │  (compliance_checked)    │  │
                            │  └───────────┬──────────────┘  │
                            │              │                  │
                            │              ▼                  │
                            │  ┌──────────────────────────┐  │
  DocuSign-BR callback      │  │  Signature Orchestrator  │◄─┼──── POST /signatures/callback
  ────────────────────►     │  │  (Domain Core)           │  │     (from DocuSign-BR)
                            │  └──────┬──────────┬────────┘  │
                            │         │          │            │
                            │         ▼          ▼            │
                            │  ┌───────────┐ ┌────────────┐  │     ┌─────────────┐
                            │  │ DocuSign  │ │ Persistence│  │     │ PostgreSQL  │
                            │  │ Client    │ │ Adapter    │──┼────►│ signature_db│
                            │  │ Adapter   │ └────────────┘  │     │ (PII encr.) │
                            │  └─────┬─────┘                 │     └─────────────┘
                            │        │       ┌────────────┐  │
                            │        │       │ Kafka      │  │
                            │        │       │ Publisher  │──┼───► signature_requested
                            │        │       │            │──┼───► contract_signed
                            │        │       └────────────┘  │     (Kafka)
                            └────────┼───────────────────────┘
                                     │
                                     ▼
                              ┌──────────────┐
                              │  DocuSign-BR │
                              │  (External)  │
                              └──────────────┘
```

## Data Flow

1. **Trigger.** A `compliance_checked` event arrives on Kafka. The consumer filters for `verdict=PASS` and discards FAIL events. The contract ID and signer metadata are extracted from the event payload.

2. **Envelope Creation.** The Signature Orchestrator builds a DocuSign-BR envelope containing the contract PDF (fetched from the document URL in the event), signing field positions, and signer identity details (name, CPF, email). The DocuSign Client Adapter sends this to the DocuSign-BR REST API.

3. **Persistence.** A `SignatureRequest` record is persisted with the DocuSign envelope ID, signer details (encrypted), and status `PENDING`. A `signature_requested` event is published to Kafka for observability.

4. **Signing.** The signer receives an email from DocuSign-BR with a link to review and sign the document. This happens entirely outside CredityFlow.

5. **Callback.** When the signer completes (or declines), DocuSign-BR sends a webhook to `POST /api/v1/signatures/callback`. The controller validates the HMAC header, then passes the event to the Orchestrator. The `SignatureStatus` is updated. If the event indicates completion, the service downloads the `SignatureCertificate` from DocuSign-BR and persists it.

6. **Completion.** A `contract_signed` event is published to Kafka, carrying the contract ID and certificate reference. Downstream, `contract-store-service` archives the signed document and `registry-orchestrator-service` begins lien registration.

## Integrations

| System | Direction | Protocol | Notes |
|---|---|---|---|
| compliance-check-service | Inbound | Kafka | `compliance_checked` events |
| DocuSign-BR | Outbound | HTTPS REST | Envelope creation, certificate download |
| DocuSign-BR | Inbound | HTTPS webhook | Signing event callbacks |
| contract-store-service | Outbound | Kafka | `contract_signed` events |
| registry-orchestrator-service | Outbound | Kafka | `contract_signed` events (fan-out) |
| PostgreSQL (signature_db) | Internal | JDBC | Encrypted PII storage |

## ADR-001: HMAC Validation for Webhook Security

**Status:** Accepted

**Context:** DocuSign-BR sends signing events to a publicly reachable callback URL. Without verification, an attacker could forge callback requests and trick the system into marking contracts as signed.

**Decision:** Validate every callback using the HMAC-SHA256 signature provided in the `X-DocuSign-Signature` header. The shared secret is stored in AWS Secrets Manager and rotated quarterly. Requests failing validation receive a 401 response and are logged with the source IP for security review.

**Consequences:** Adds a small computational overhead per callback (negligible). Requires secret rotation coordination with DocuSign-BR support. Provides strong assurance that callbacks are authentic.

## ADR-002: Column-Level Encryption for PII

**Status:** Accepted

**Context:** The service stores signer PII (name, CPF, email, phone) required by DocuSign-BR. Brazilian LGPD regulations and internal security policy require encryption of PII at rest beyond what disk-level encryption provides.

**Decision:** Apply AES-256-GCM column-level encryption on all PII fields in the `signature_requests` table using a JPA attribute converter. The encryption key is loaded from AWS Secrets Manager at startup and cached in memory. Encrypted columns use `bytea` type in PostgreSQL.

**Consequences:** PII fields cannot be queried directly via SQL (they are opaque bytes). Lookups by signer CPF require a separate indexed hash column (`cpf_hash`). Key rotation requires a background migration job to re-encrypt existing rows, which is run during maintenance windows.
