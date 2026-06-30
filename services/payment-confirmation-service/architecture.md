# Architecture: payment-confirmation-service

## Component Diagram

```
                          +----------------------------+
                          | BancoParceiro              |
                          | (partner bank)             |
                          +---+--------------------+---+
                              |                    ^
                    Webhook (POST)          Polling (GET)
                              |                    |
                              v                    |
+------------------------------------------------------------------------+
|                 payment-confirmation-service                            |
|                                                                        |
|  +---------------------+    +------------------------+                 |
|  | WebhookReceiver     |    | PollingScheduler       |                 |
|  | (REST endpoint)     |    | (every 5 min)          |                 |
|  +----------+----------+    +----------+-------------+                 |
|             |                          |                               |
|             +------------+-------------+                               |
|                          |                                             |
|                          v                                             |
|             +------------------------+                                 |
|             | ConfirmationService    |                                 |
|             +---+--------+-----------+                                 |
|                 |        |                                             |
|                 v        v                                             |
|  +-------------------+  +------------------------+                     |
|  | ReconciliationEngine|  | EventPublisher        |                    |
|  | (amount matching)  |  | (Kafka)               |                    |
|  +-------------------+  +------------------------+                     |
|                                                                        |
|                          +------------------+                          |
|                          | PostgreSQL       |                          |
|                          | disbursement_db  |                          |
|                          +------------------+                          |
+------------------------------------------------------------------------+
              |                                    ^
   bank_transfer_confirmed (Kafka)      bank_transfer_initiated (Kafka)
              |                                    |
              v                                    |
+----------------------------+       +----------------------------+
| disbursement-service       |       | bank-transfer-integration  |
+----------------------------+       +----------------------------+
```

## Data Flow

1. **Transfer Tracking Begins** -- The service consumes a `bank_transfer_initiated` event from the `credityflow.disbursement` Kafka topic. This event, published by `bank-transfer-integration`, contains the transfer ID, expected amount, and bank protocol. A tracking record is created in `disbursement_db` with status `AWAITING_CONFIRMATION`.
2. **Webhook Confirmation (primary path)** -- BancoParceiro sends a POST webhook to `/api/v1/confirmations/webhook` when the transfer is settled. The `WebhookReceiver` validates the HMAC-SHA256 signature, parses the payload, and delegates to `ConfirmationService`. The confirmation is persisted as a `PaymentConfirmation` record with `source=WEBHOOK`.
3. **Polling Fallback (secondary path)** -- The `PollingScheduler` runs every 5 minutes and identifies transfers that have been in `AWAITING_CONFIRMATION` status for longer than 30 minutes. For each stale transfer, it calls the BancoParceiro transfer status API. If the transfer has been settled, a `PaymentConfirmation` is created with `source=POLLING`.
4. **Reconciliation** -- Regardless of confirmation source, the `ReconciliationEngine` compares the confirmed amount against the expected amount from the original transfer. A `ReconciliationRecord` is created with status `MATCHED` (amounts equal within R$0.01 tolerance) or `MISMATCHED` (discrepancy detected). Mismatches trigger a critical alert.
5. **Event Publication** -- Once confirmation and reconciliation are complete, the `EventPublisher` publishes a `bank_transfer_confirmed` event to the `credityflow.disbursement` Kafka topic. This event includes the confirmation status, confirmed amount, reconciliation result, and confirmation source.

## Integrations

| System | Direction | Protocol | Purpose |
|---|---|---|---|
| bank-transfer-integration | Inbound | Kafka event | Receives bank_transfer_initiated to start tracking |
| BancoParceiro | Inbound | Webhook (HTTPS) | Receives payment confirmation callbacks |
| BancoParceiro | Outbound | REST API | Polls transfer status as fallback |
| disbursement-service | Outbound | Kafka event | Publishes bank_transfer_confirmed |
| PostgreSQL (disbursement_db) | Internal | JDBC | Confirmations and reconciliation records |

## Architecture Decision Records

### ADR-037-1: Dual Confirmation Strategy (Webhook + Polling)

**Status:** Accepted
**Context:** BancoParceiro provides a webhook mechanism for real-time transfer confirmation. However, production experience revealed that webhooks are not 100% reliable: approximately 2-3% of webhooks are delayed beyond 30 minutes, and during BancoParceiro incident windows, webhook delivery can halt entirely. Relying solely on webhooks would leave transfers in an unconfirmed state indefinitely, blocking the disbursement lifecycle and creating reconciliation gaps.
**Decision:** Implement a dual confirmation strategy. The webhook path is the primary mechanism, providing near-real-time confirmation (typically within 1-5 minutes). A polling scheduler acts as a safety net, querying the BancoParceiro status API for any transfer that has not been confirmed via webhook within 30 minutes. Both paths converge in the same `ConfirmationService`, which handles idempotency (a transfer can only be confirmed once, regardless of source).
**Consequences:** 100% of transfers are eventually confirmed, even during BancoParceiro webhook outages. The polling fallback adds load to the BancoParceiro API (rate-limited to 100 requests/minute), but under normal conditions only 2-3% of transfers require polling. The `confirmation_source` field in `PaymentConfirmation` enables monitoring of webhook reliability over time. If the webhook failure rate increases, the polling interval can be reduced. The dual strategy adds implementation complexity but eliminates a critical gap in the disbursement lifecycle.
