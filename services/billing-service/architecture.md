# Billing Service

**svc-38 | Collections Domain | Kotlin/Spring Boot | PostgreSQL (billing_db)**

---

## Purpose

The Billing Service is the financial heartbeat of CredityFlow's post-origination lifecycle. It generates installment schedules from disbursed loan contracts, tracks incoming payments, and detects overdue installments that trigger downstream collection workflows.

This service does NOT process payments -- it records payment confirmations received from the payment gateway and reconciles them against expected installments.

## Architecture Overview

### Core Modules

1. **Schedule Generator** (`InstallmentScheduleService`)
   - Receives `LoanDisbursed` events from the origination domain.
   - Computes the full amortization schedule (SAC or Price table, depending on product configuration).
   - Persists each installment as a row in `billing_db.installments` with status `PENDING`, due date, principal, interest, and fees.

2. **Payment Reconciler** (`PaymentReconciliationService`)
   - Consumes `PaymentConfirmed` events from the payment gateway integration.
   - Matches payments to installments using contract ID + installment number.
   - Updates installment status to `PAID`, `PARTIALLY_PAID`, or `OVERPAID`.
   - Handles early payments and prepayment discount calculations.

3. **Overdue Detector** (Cron Job)
   - Runs daily at 01:00 UTC via Spring `@Scheduled`.
   - Queries installments where `due_date < today AND status = PENDING`.
   - Transitions matched installments to `OVERDUE` and publishes `InstallmentOverdue` events.
   - Calculates late fees (mora + multa) according to contract terms.

4. **REST API**
   - `GET /v1/contracts/{contractId}/schedule` -- full installment schedule.
   - `GET /v1/contracts/{contractId}/balance` -- outstanding balance with real-time interest accrual.
   - `POST /v1/contracts/{contractId}/simulate-prepayment` -- early payoff simulation.

### Database Schema (Key Tables)

| Table | Purpose |
|-------|---------|
| `installments` | One row per installment per contract. Core billing table. |
| `payments` | Payment records linked to installments. |
| `late_fees` | Computed late fee records attached to overdue installments. |
| `prepayment_simulations` | Cached simulation results (TTL-based eviction). |

## Data Flow

```
LoanDisbursed (SNS) ----> SQS ----> Schedule Generator ----> billing_db
                                                                  |
PaymentConfirmed (SNS) -> SQS ----> Payment Reconciler ------->--+
                                                                  |
Cron (daily 01:00 UTC) -----------> Overdue Detector -----------+
                                          |
                                          v
                                    SNS: InstallmentOverdue
                                          |
                              +-----------+-----------+
                              v                       v
                       dunning-service      collections-contact-service
```

## Integration Patterns

- **Inbound events:** `LoanDisbursed` from origination, `PaymentConfirmed` from payment gateway. Both consumed via SQS with at-least-once delivery.
- **Outbound events:** `InstallmentOverdue`, `InstallmentPaid`, `ContractFullyPaid`. Published to SNS for fan-out.
- **Synchronous APIs:** Called by the customer portal BFF and backoffice BFF for schedule display and balance queries.
- **Idempotency:** Payment reconciliation uses the payment gateway's transaction ID as an idempotency key. Duplicate `PaymentConfirmed` events are safely ignored.

## ADR Notes

### ADR-038-001: Daily cron for overdue detection instead of event-driven

**Context:** An event-driven approach (scheduling a delayed message per installment) was considered. With thousands of active contracts, this would mean thousands of scheduled messages to manage, each needing cancellation logic if paid early.

**Decision:** Use a simple daily cron job that scans for overdue installments in batch. The job runs at 01:00 UTC to align with the Brazilian banking day boundary.

**Consequences:** Overdue detection has up to ~24 hours of latency, which is acceptable for the collections workflow. The cron job is simpler to monitor and debug than distributed delayed messages. If the job fails, it can be safely re-run (idempotent state transitions).

### ADR-038-002: Store full amortization schedule at disbursement time

**Context:** An alternative was to compute installments on-the-fly. This would save storage but make historical queries and auditing harder.

**Decision:** Materialize the full schedule at loan disbursement. Each installment is a first-class entity that can be individually tracked, modified (for renegotiation), and audited.

**Consequences:** More storage usage but dramatically simpler query patterns. Renegotiation flows can update individual installments without recomputing the entire schedule. Enables straightforward reporting on aging and portfolio health.
