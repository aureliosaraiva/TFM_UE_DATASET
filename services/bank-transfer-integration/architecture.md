# Architecture: bank-transfer-integration

## Component Diagram

```
                     +----------------------------+
                     |   disbursement-service     |
                     +-------------+--------------+
                                   |
                    disbursement_requested (Kafka)
                                   |
                                   v
+------------------------------------------------------------------------+
|                    bank-transfer-integration                           |
|                                                                        |
|  +----------------+    +------------------------+                      |
|  | EventConsumer  |--->| TransferService         |                     |
|  | (Kafka)        |    +---+------------------+-+                      |
|  +----------------+        |                  |                        |
|                            v                  v                        |
|  +---------------------+ +------------------+ +-----------------+     |
|  | TransferController  | | TransferMethod   | | EventPublisher  |     |
|  | (REST API)          | | Selector         | | (Kafka)         |     |
|  +---------------------+ +--------+---------+ +-----------------+     |
|                                    |                                   |
|                         +----------v-----------+                       |
|                         | BancoParceiroClient   |                      |
|                         | (OAuth2 HTTP client)  |                      |
|                         +----------+-----------+                       |
|                                    |           +------------------+    |
|                                    |           | PostgreSQL       |    |
|                                    |           | disbursement_db  |    |
|                                    |           +------------------+    |
+------------------------------------------------------------------------+
                                     |
                                     v (HTTPS + OAuth2, IP whitelist)
                          +------------------------+
                          |    BancoParceiro API    |
                          |    (TED / PIX)          |
                          +------------------------+

                    bank_transfer_initiated (Kafka)
                                     |
                                     v
                     +-------------------------------+
                     | payment-confirmation-service  |
                     +-------------------------------+
```

## Data Flow

1. **Disbursement Requested** -- The service consumes a `disbursement_requested` event from the `credityflow.disbursement` Kafka topic.
2. **Method Selection** -- The `TransferMethodSelector` evaluates:
   - If the recipient has a `pix_key` and the amount is within PIX limits (R$1,000,000), use PIX.
   - If it is outside TED banking hours (06:30-17:00 BRT on business days) and PIX is available, prefer PIX.
   - Otherwise, use TED.
3. **Transfer Submission** -- The `BancoParceiroClient` submits the transfer to BancoParceiro API via HTTPS with OAuth2 authentication. The request originates from a whitelisted static IP.
4. **Acknowledgment** -- BancoParceiro returns a bank protocol number. The `BankTransfer` record is updated to `ACKNOWLEDGED`.
5. **Event Published** -- A `bank_transfer_initiated` event is published to Kafka with the bank protocol and transfer details for confirmation tracking.

## Integrations

| System | Direction | Protocol | Purpose |
|---|---|---|---|
| disbursement-service | Inbound | Kafka event | Receives disbursement_requested for transfer execution |
| BancoParceiro API | Outbound | REST + OAuth2 | Executes TED and PIX bank transfers |
| payment-confirmation-service | Outbound | Kafka event | Publishes bank_transfer_initiated for confirmation |
| PostgreSQL (disbursement_db) | Internal | JDBC | Persists transfer data and bank account info |

## Architecture Decision Records

### ADR-036-1: Transfer Method Selection Strategy

**Status:** Accepted
**Context:** BancoParceiro supports both TED and PIX transfers. PIX is instant but has regulatory amount limits. TED is traditional but restricted to banking hours. The service needs to automatically select the best method for each transfer.
**Decision:** Implement a `TransferMethodSelector` that applies the following priority logic: (1) if PIX key is available and amount is within the R$1,000,000 PIX limit, use PIX; (2) if outside TED banking hours and PIX is available, use PIX to avoid next-day processing; (3) otherwise, use TED. The selection logic is configurable via application properties for threshold and time window adjustments.
**Consequences:** Most transfers will use PIX when available, providing faster fund delivery to borrowers. TED remains the fallback for high-value transfers and recipients without PIX keys. The selection logic may need updates as Central Bank regulations change.

### ADR-036-2: Static Egress IP for BancoParceiro Communication

**Status:** Accepted
**Context:** BancoParceiro requires all API requests to originate from pre-registered IP addresses as an additional security layer beyond OAuth2 authentication.
**Decision:** Deploy the service behind a NAT gateway with static egress IPs registered with BancoParceiro. The Kubernetes cluster uses a dedicated egress configuration for the `disbursement` namespace to ensure consistent source IPs.
**Consequences:** Infrastructure changes that affect egress IPs (cluster migration, multi-region deployment) require coordination with BancoParceiro to update the IP whitelist. This introduces an operational dependency and limits deployment flexibility.
