# Case Management Service

```
Service ID:  svc-47
Domain:      Backoffice
Stack:       Kotlin / Spring Boot 3.x
Database:    PostgreSQL (case_management_db)
```

## Overview

The Case Management Service provides a unified abstraction for tracking work items across CredityFlow's operational domains. A "case" is a container that groups related activities -- an appraisal review, a dunning escalation, a compliance investigation, a legal proceeding -- into a single entity that backoffice agents can track, assign, and resolve.

The service does not own the underlying domain data. It holds metadata: case type, status, priority, assignee, timeline, and links to the domain-specific records in other services. It is the operational backbone of the backoffice.

## Why a Cross-Domain Case Service?

Without it, each domain would implement its own task/queue management. Agents working across domains (e.g., a collections manager who also handles compliance alerts) would need to juggle multiple UIs and mental models. The case management service provides:

- A single inbox for all operational work.
- Consistent priority scoring across domains.
- SLA tracking with escalation rules.
- Full activity history for any case.

## Core Concepts

**Case:** The central entity. Has a type, status, priority, assignee, SLA deadline, and a set of references to external entities (contract ID, customer ID, appraisal ID, etc.).

**Case Type:** Defines the workflow. Types include `APPRAISAL_REVIEW`, `DUNNING_ESCALATION`, `COMPLIANCE_ALERT`, `LEGAL_CASE`, `CUSTOMER_COMPLAINT`, and `MANUAL_TASK`.

**Activity Log:** An append-only log of everything that happens on a case: status changes, reassignments, comments, linked documents, external events.

**SLA Policy:** Per-case-type configuration defining response time and resolution time targets. Breaches trigger escalation events.

## Components

| Component | Responsibility |
|-----------|---------------|
| `CaseController` | REST API for CRUD, search, and bulk operations |
| `CaseEventListener` | Consumes domain events that create or update cases |
| `SlaMonitor` | Cron job (every 15 min) checking for SLA breaches |
| `AssignmentEngine` | Auto-assigns cases based on team capacity and skills |
| `ActivityLogger` | Records all case mutations as immutable activity entries |
| `SearchService` | Full-text and filtered search over cases (PostgreSQL full-text + GIN indexes) |

## Data Flow

### Case Creation (Event-Driven)

```
Domain Service (e.g., dunning-service)
        |
        | DunningEscalated event (SNS/SQS)
        v
CaseEventListener
        |
        v
AssignmentEngine --> determines assignee
        |
        v
case_management_db (case created, status: OPEN)
        |
        v
SNS: CaseCreated event --> backoffice-bff (queue refresh)
```

### Case Resolution (API-Driven)

```
Backoffice BFF
        |
        | PUT /v1/cases/{id}/resolve
        v
CaseController
        |
        v
ActivityLogger --> records resolution with notes
        |
        v
case_management_db (status: RESOLVED)
        |
        v
SNS: CaseResolved event --> originating domain service
```

## Integration Patterns

**Inbound events:** The service listens for events from multiple domains:
- `AppraisalCompleted` (with review flag) from appraisal services.
- `DunningEscalated` from dunning-service.
- `LegalCaseCreated` from legal-collections-service.
- `ComplianceAlertRaised` from audit-service.

Each event type maps to a case creation handler that extracts relevant metadata and determines priority.

**Outbound events:** `CaseCreated`, `CaseUpdated`, `CaseResolved`, `SlaBreach`. Consumed primarily by the backoffice BFF for real-time queue updates.

**REST API:** The backoffice BFF is the primary consumer. Supports filtering by assignee, type, status, priority, and SLA status. Pagination and sorting are essential given the volume of cases.

**No direct domain queries:** The case service stores references (IDs) to domain entities but does not fetch domain data. The backoffice BFF is responsible for enriching case data with domain details when rendering the UI.

## ADR

### ADR-047-001: Generic case model over domain-specific task services

**Context:** Each domain team initially wanted its own task tracking. This would have resulted in 4-5 separate queue/task systems with inconsistent UX and no cross-domain visibility.

**Decision:** Build a generic case management service with configurable case types. Domain-specific behavior is encoded in case type configuration, not in code.

**Consequences:** Unified agent experience across all operational domains. New case types can be added without code changes (configuration-driven). The tradeoff is that domain-specific workflows sometimes feel constrained by the generic model -- addressed by allowing custom metadata fields per case type via a `jsonb` column.

### ADR-047-002: PostgreSQL full-text search over Elasticsearch

**Context:** Case search needs to support free-text queries across case notes, customer names, and contract IDs. Elasticsearch was considered for this purpose.

**Decision:** Use PostgreSQL's built-in full-text search with GIN indexes. The case volume (tens of thousands, not millions) and query patterns (mostly filtered + keyword) are well within PostgreSQL's capabilities.

**Consequences:** No additional infrastructure to operate. Search quality is adequate for internal use. If case volume grows 100x or search requirements become more sophisticated (fuzzy matching, relevance scoring), migration to Elasticsearch remains an option. The audit service already operates an ES cluster that could be shared.
