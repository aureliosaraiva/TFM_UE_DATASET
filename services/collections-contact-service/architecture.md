# Collections Contact Service -- Architecture

| Field | Detail |
|-------|--------|
| **Service ID** | svc-40 |
| **Domain** | Collections |
| **Tech** | Kotlin, Spring Boot 3.x |
| **Database** | PostgreSQL (`collections_db`) |
| **Deployment** | AWS EKS, 2 replicas (auto-scales to 6) |

## What It Does

The Collections Contact Service manages outbound contact attempts with customers who have overdue obligations. It maintains a contact queue, tracks attempt history, enforces regulatory contact rules (time-of-day restrictions, maximum attempts per period), and records outcomes.

This service does not send messages directly. It orchestrates *when* and *how* to contact a customer, then delegates actual delivery to the notification service. Think of it as the "contact strategy brain" for collections.

## Components

**ContactQueueManager** -- Maintains a prioritized queue of customers to contact. Priority is calculated from days-past-due, outstanding amount, and customer risk score. Receives `ScheduleContact` commands from the dunning service.

**ContactRuleEngine** -- Enforces contact policies before any attempt:
- No calls before 08:00 or after 20:00 local time (Brazilian Consumer Defense Code).
- Maximum 3 attempts per day per customer across all channels.
- Minimum 2-hour gap between attempts.
- Respect opt-out flags and "do not contact" periods.

**AttemptExecutor** -- When the rule engine approves a contact, selects the channel (phone, SMS, email, WhatsApp) based on the configured contact strategy and past response rates. Dispatches to the notification service with the appropriate template code and channel.

**OutcomeRecorder** -- Receives outcome callbacks: answered, not-answered, promise-to-pay, refused, wrong-number. Updates the contact record and publishes `ContactCompleted` events consumed by the dunning service.

**REST API** -- Exposes endpoints for the backoffice BFF to view contact history, manually schedule contacts, and override contact strategies.

## Data Flow

```
dunning-service                                    notification-service
     |                                                     ^
     | ScheduleContact (SQS)                              |
     v                                                     |
ContactQueueManager --> ContactRuleEngine --> AttemptExecutor
                                                     |
                                                     v
                                              collections_db
                                              (contact_attempts)
                                                     |
                                                     v
                                              ContactCompleted (SNS)
                                                     |
                                                     v
                                              dunning-service
```

## Integration Patterns

- **Inbound:** Asynchronous commands from dunning-service (`ScheduleContact`) via SQS. Backoffice manual triggers via REST.
- **Outbound to notification-service:** Synchronous REST call to `POST /v1/notifications/send`. The synchronous call is intentional here -- the service needs the notification ID for tracking delivery status.
- **Outbound events:** `ContactCompleted` published to SNS for the dunning service to advance its campaign state machine.
- **Customer data:** Fetches customer contact details from the customer service via REST (cached 15 min). Phone number validation is done by the customer service; this service trusts the data it receives.

## ADR

### ADR-040-001: Centralized contact rule enforcement

**Context:** Contact rules could be enforced at multiple levels -- in the dunning service, in this service, or in the notification service. Distributing the rules risked inconsistent enforcement and regulatory violations.

**Decision:** Centralize all contact compliance rules in the Collections Contact Service. The dunning service simply requests a contact; this service decides if, when, and how it happens.

**Consequences:** Single point of audit for regulatory compliance. The dunning service's campaign logic stays simple -- it does not need to know about time-of-day restrictions. If rules change (e.g., new BACEN regulation), only this service is updated.

### ADR-040-002: Channel selection based on historical response rates

**Context:** Early versions used a static channel preference list (phone first, then SMS, then email). This ignored that some customers respond better to WhatsApp while others prefer SMS.

**Decision:** Implement a simple scoring model that ranks channels by the customer's historical response rate. Falls back to the product-level default if no history exists.

**Consequences:** Improved contact success rates (~18% improvement in pilot). Requires storing per-customer channel response history. The model is rule-based, not ML -- straightforward to explain and audit.
