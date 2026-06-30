# document-validation-service

## Overview

The document-validation-service applies business rules to classified documents to determine whether they are acceptable for a loan proposal. It checks things like: Is the identity document expired? Is the OCR readability score above the minimum threshold? Were all required fields successfully extracted? Does the name on the document match the customer profile?

This is the quality gate in the document pipeline. Documents that pass validation move to final storage; those that fail trigger a re-upload request to the customer.

**Stack**: Kotlin/Spring Boot, PostgreSQL | **Team**: team-documents | **Domain**: Document Management

## Getting Started

```bash
docker-compose up -d postgres
./gradlew bootRun --args='--spring.profiles.active=local'
```

The local profile uses a seeded set of validation rules. In production, rules are managed by the operations team through an internal admin tool.

```bash
./gradlew test
./gradlew integrationTest
```

## Architecture

The service is event-driven on the input side (consumes `documents_classified` from SQS) and publishes results via SNS (`documents_validated`). It also has a synchronous dependency on customer-service for cross-reference validations.

Validation rules are stored in PostgreSQL and loaded into memory on startup, refreshing every 5 minutes. Each rule implements a common `ValidationRule` interface, making it straightforward to add new rule types. Current rule types:

- **Expiry check**: Document date fields compared against configurable thresholds
- **Readability check**: OCR confidence score must exceed a minimum per document type
- **Completeness check**: Required fields for the document category must all be present
- **Cross-reference check**: Name and CPF extracted from the document are compared against the customer profile via customer-service API

See [architecture.md](architecture.md) for the validation flow and ADR on rule engine design.

## API Reference

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/validate` | Validate a document (accepts document_id, category, extracted fields) |
| `GET` | `/api/v1/validation-rules` | List active rules, optionally filtered by document category |

The POST endpoint is primarily invoked internally via event processing, but it is also available for ad-hoc validation by internal tools.

## Configuration

| Variable | Description |
|----------|-------------|
| `DATABASE_URL` | PostgreSQL connection for document_validation_db |
| `SQS_QUEUE_URL` | Queue for documents_classified events |
| `SNS_TOPIC_ARN` | Topic for documents_validated events |
| `CUSTOMER_SERVICE_URL` | Base URL for customer-service (used in cross-reference rules) |
| `RULE_REFRESH_INTERVAL_SECONDS` | How often to reload rules from DB (default: 300) |

## Deployment

Standard EKS Helm deployment.

- **CPU**: 500m / 1000m
- **Memory**: 512Mi / 1024Mi
- **Replicas**: 2-3

The service is relatively lightweight. The main performance concern is the synchronous call to customer-service during cross-reference validation, which is protected by a circuit breaker (Resilience4j, 3s timeout).

Database migrations via Flyway. Seed data for default validation rules is included in migration scripts.

## Monitoring

Metrics:
- `validation_total` by category and result (pass/fail)
- `validation_rule_failure_total` by rule type and category
- `cross_reference_call_duration_seconds`

When the overall failure rate for a specific document category exceeds 40%, an alert fires. This usually indicates an upstream problem (OCR quality degraded, classification errors) rather than a rule configuration issue.

Dashboard: "Validation Results" shows pass/fail trends and identifies which rules are rejecting the most documents.

## Known Issues

- **Rule versioning via API**: Rules can only be updated through direct database access or the admin tool. There is no versioned REST API for rule management, which makes audit trails harder to maintain.
- **Cross-reference latency**: The synchronous call to customer-service adds 50-200ms to validation. If customer-service is slow or down, the circuit breaker opens and cross-reference rules are skipped (with a warning flag on the result).
- **No forgery detection**: The service validates data quality and consistency but cannot detect forged or tampered documents. That capability would require integration with a specialized fraud detection provider.
