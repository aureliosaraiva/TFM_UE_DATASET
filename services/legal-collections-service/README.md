# legal-collections-service

**svc-42 | Collections Domain | team-collections | Production**

## Overview

The legal-collections-service handles the most serious stage of debt recovery: formal legal proceedings. When amicable collection efforts (dunning, negotiation) have been exhausted, this service takes over by filing debt protests with notary offices and initiating judicial collection actions through the court system.

Because it deals with legal instruments, this service has stricter compliance requirements than most. It stores PII alongside financial data, maintains a full audit log of every action, and integrates with external legal systems that have their own unpredictable latency and availability characteristics.

## Getting Started

### Prerequisites

- JDK 17+
- Docker Compose (PostgreSQL, Kafka)
- Gradle 8.x
- Access credentials for Notary Office API and Court Filing System (for integration testing)

### Local Development

```bash
docker-compose up -d postgres kafka

# The service uses a mock legal-api profile for local development
./gradlew bootRun --args='--spring.profiles.active=local,mock-legal-api'
```

Default port: `8083`. Health check: `http://localhost:8083/actuator/health`

### Environment Variables

| Variable | Required | Description |
|---|---|---|
| `SPRING_DATASOURCE_URL` | Yes | JDBC URL for legal_collections_db |
| `SPRING_KAFKA_BOOTSTRAP_SERVERS` | Yes | Kafka brokers |
| `LEGAL_NOTARY_API_URL` | Yes | Notary Office API base URL |
| `LEGAL_NOTARY_API_KEY` | Yes | API key for notary integration |
| `LEGAL_COURT_FILING_URL` | Yes | Court Filing System endpoint |
| `LEGAL_COURT_FILING_CLIENT_CERT` | Yes | mTLS client certificate path |
| `LEGAL_CONTACT_SERVICE_URL` | Yes | collections-contact-service base URL |

## Architecture

The service is structured around two parallel flows -- **protest** and **judicial** -- that share a common `LegalProceeding` aggregate root. Each flow has its own adapter for the corresponding external system, wrapped in circuit breakers to handle unavailability gracefully.

**Core entities:**
- `LegalProceeding` -- the umbrella case that tracks the overall legal recovery effort
- `ProtestRecord` -- notary protest details (filing date, notary ID, resolution status)
- `JudicialAction` -- court case details (case number, jurisdiction, hearing schedule)

All write operations are recorded in an append-only `audit_log` table. The database (`legal_collections_db`) contains both PII and financial data, so access is restricted via row-level security policies.

Full details in [architecture.md](./architecture.md).

## API Reference

### POST /api/v1/legal/initiate

Initiates a legal proceeding. The `type` field determines whether it is a protest or judicial action.

**Request:**
```json
{
  "proposal_id": "prop-789",
  "type": "PROTEST",
  "reason": "Borrower unresponsive after 90 days"
}
```

**Response 201:**
```json
{
  "proceeding_id": "proc-100",
  "type": "PROTEST",
  "status": "INITIATED",
  "created_at": "2026-03-29T09:00:00Z"
}
```

Returns `409 Conflict` if a proceeding already exists for the proposal.

### GET /api/v1/legal/{proposal_id}/status

Returns the current status and full history of legal proceedings for a proposal.

**Response 200:**
```json
{
  "proposal_id": "prop-789",
  "proceedings": [
    {
      "proceeding_id": "proc-100",
      "type": "PROTEST",
      "status": "FILED",
      "filed_at": "2026-03-29T09:15:00Z",
      "history": [
        { "status": "INITIATED", "at": "2026-03-29T09:00:00Z" },
        { "status": "FILED", "at": "2026-03-29T09:15:00Z" }
      ]
    }
  ]
}
```

## Configuration

- **`legal.protest.min-overdue-days`** -- Minimum days overdue before a protest can be filed (default: 60).
- **`legal.judicial.min-overdue-days`** -- Minimum days overdue before judicial action (default: 120).
- **`legal.protest.min-amount`** -- Minimum debt amount for protest eligibility.
- **`legal.circuit-breaker.*`** -- Resilience4j circuit breaker settings for external integrations.

Jurisdiction-specific rules are loaded from the `jurisdiction_config` table at startup and refreshed every 15 minutes.

## Deployment

```bash
./gradlew bootBuildImage --imageName=credityflow/legal-collections-service:latest

helm upgrade --install legal-collections deploy/helm \
  --namespace collections \
  --values deploy/helm/values-staging.yaml
```

**Important:** The Court Filing System integration uses mTLS. Certificates must be mounted as Kubernetes secrets and referenced in `LEGAL_COURT_FILING_CLIENT_CERT`.

Database migrations via Flyway. Because this database contains PII, migration scripts must be reviewed by the security team before merging.

## Monitoring

| Metric | Description |
|---|---|
| `legal_proceedings_initiated_total` | Proceedings started, labeled by type |
| `legal_protests_filed_total` | Protests submitted to notary offices |
| `legal_judicial_actions_filed_total` | Court filings submitted |
| `legal_proceeding_resolution_duration_days` | Time from initiation to resolution |

**Critical alerts:**
- *NotaryAPIUnavailable* -- more than 5 errors in 10 minutes from the notary integration.
- *HighPendingProceedings* -- over 1,000 proceedings stuck in non-terminal states.

Audit logs can be queried through the internal audit dashboard (Kibana, index `legal-collections-audit-*`).

## Known Issues

1. **Notary API downtime:** The notary office API has scheduled maintenance windows (typically weekends). Protests initiated during these windows queue up and are retried automatically, but SLA tracking should account for the delay.
2. **Jurisdiction coverage:** Not all jurisdictions are configured yet. Attempting to initiate a proceeding in an unconfigured jurisdiction returns a `400` with a descriptive error. The team is expanding coverage incrementally.
3. **PII exposure risk:** Because this service stores sensitive personal data alongside legal records, any new endpoint must undergo a privacy impact assessment before deployment. See the data classification policy for details.
