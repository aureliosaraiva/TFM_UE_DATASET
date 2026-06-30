# Customer Service (svc-03)

**Domain:** Onboarding
**Stack:** Kotlin / Spring Boot
**Database:** PostgreSQL (`customer_db`)
**Deployment:** AWS EKS

## Component Overview

The Customer Service is the canonical source of truth for customer identity within CredityFlow. It owns all personally identifiable information (PII) and serves as the central lookup endpoint for 15+ downstream services across every domain in the platform. Given its position at the heart of the architecture, this service is designed for high availability and low-latency reads.

Internally, the service is structured around three main modules:

- **Customer Profile API** -- exposes RESTful endpoints for CRUD operations on customer records. All mutations are guarded by field-level validation and idempotency keys to prevent duplicate registrations during the onboarding flow.
- **PII Vault** -- a logical layer that encrypts sensitive fields (CPF, date of birth, phone number) at rest using AWS KMS envelope encryption. Decryption happens on read, scoped to the calling service's IAM role and purpose.
- **Event Publisher** -- after every create or update, a domain event (`CustomerCreated`, `CustomerUpdated`) is published to the `customer-events` SNS topic so that dependent services can react without polling.

The database schema is deliberately narrow: a `customers` table, a `customer_addresses` table, and a `customer_contacts` table. Foreign keys are internal only; downstream services reference customers by `customer_id` (UUID).

## Data Flow

1. The onboarding frontend submits customer data via the API Gateway.
2. Customer Service validates the payload, checks for duplicates (CPF uniqueness constraint), and persists the record.
3. A `CustomerCreated` event is emitted to SNS, fanning out to SQS subscriptions owned by Proposal Service, Fraud Decision Service, and others.
4. Read requests from other services hit a read-replica-backed GET endpoint. A lightweight Redis cache (TTL 5 min) sits in front of the replica for burst traffic during peak onboarding hours.

Data leaves the service in two shapes: full profile (restricted to services with elevated IAM permissions) and masked profile (default, with PII fields redacted).

## Integration Patterns

- **Synchronous:** REST API consumed by virtually every service that needs customer context. Rate-limited per caller using Spring Cloud Gateway token buckets.
- **Asynchronous:** SNS/SQS fan-out for lifecycle events. Consumers include `proposal-service`, `fraud-decision-service`, `credit-decision-service`, and the analytics pipeline.
- **Database:** No shared database. Other services must go through the API. This is a hard architectural boundary.

## ADR-001: Centralized PII Store vs. Distributed Ownership

**Context:** Early designs considered letting each domain store its own copy of customer PII (e.g., fraud stores name + CPF, credit stores income-related fields). This would reduce coupling but create data consistency risks and multiply the compliance surface for LGPD (Brazil's data protection law).

**Decision:** All PII is owned exclusively by `customer-service`. Other services store only the `customer_id` foreign reference and fetch PII on demand or receive it in event payloads (encrypted).

**Consequences:**
- Single point of compliance for data subject access requests (DSARs) and right-to-deletion.
- Increased runtime coupling: if `customer-service` is down, services that need PII in real-time are degraded.
- Mitigated by the Redis read cache and read replicas, which provide resilience for GET operations even during brief outages.

## ADR-002: Envelope Encryption with AWS KMS

**Context:** Storing PII in plaintext in PostgreSQL was flagged during the initial security review. The team evaluated three options: application-level AES encryption, PostgreSQL TDE (transparent data encryption), and AWS KMS envelope encryption.

**Decision:** Use AWS KMS envelope encryption. A data encryption key (DEK) is generated per customer record, encrypted by a KMS customer master key (CMK), and stored alongside the ciphertext. Decryption requires an IAM-authorized KMS call.

**Consequences:**
- Strong key rotation and audit trail via CloudTrail.
- Slight latency overhead on reads (~2-5ms per KMS call), acceptable given caching.
- Operational complexity: key policy management must be coordinated with the platform team whenever a new service needs PII access.
