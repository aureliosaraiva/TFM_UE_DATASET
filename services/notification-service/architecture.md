# Architecture: notification-service

## Component Diagram

```
+-------------------+  +-------------------+  +-------------------+
| credit-decision-  |  | disbursement-     |  | collections-      |
| service            |  | service            |  | contact-service    |
+--------+----------+  +--------+----------+  +--------+----------+
         |                       |                       |
         +----------+------------+-----------------------+
                    |
           Kafka (25+ event types)
                    |
                    v
+------------------------------------------------------------------------+
|                    notification-service                                 |
|                                                                        |
|  +---------------------+    +------------------------+                 |
|  | EventConsumer       |--->| NotificationRouter     |                 |
|  | (Kafka, 25+ topics) |    +---+----+----+----+-----+                 |
|  +---------------------+        |    |    |    |                       |
|                                 v    v    v    v                       |
|  +----------+ +----------+ +----------+ +----------+                   |
|  | Email    | | SMS      | | Push     | | WhatsApp |                   |
|  | Sender   | | Sender   | | Sender   | | Sender   |                   |
|  | (SES)    | | (SNS)    | | (FCM)    | | (Twilio) |                   |
|  +----------+ +----------+ +----------+ +----------+                   |
|                                                                        |
|  +---------------------+    +------------------------+                 |
|  | PreferenceService   |    | DeliveryTracker        |                 |
|  | (channel prefs)     |    | (status + retry)       |                 |
|  +---------------------+    +------------------------+                 |
|                                                                        |
|                          +------------------+                          |
|                          | PostgreSQL       |                          |
|                          | notification_db  |                          |
|                          +------------------+                          |
+------------------------------------------------------------------------+
          |
          | REST (render template)
          v
+----------------------------+
| notification-template-     |
| service                    |
+----------------------------+
```

## Data Flow

1. **Event Consumption** -- The `EventConsumer` subscribes to 25+ Kafka topics spanning all CredityFlow domains (e.g., `credit_decision_approved`, `disbursement_requested`, `contract_signed`, `payment_overdue`). Each event type is mapped to one or more notification templates via configuration.
2. **Channel Resolution** -- `NotificationRouter` resolves the customer's channel preferences via `PreferenceService`. Preferences are stored per customer in `notification_db` and respect opt-out requests. If a customer has not set preferences, the default channel priority is: push > email > SMS > WhatsApp.
3. **Template Rendering** -- The router calls `notification-template-service` via REST to render the notification content. The event payload is passed as template variables. The template service returns the rendered subject and body for the selected channel.
4. **Dispatch** -- The rendered message is dispatched through the appropriate channel sender:
   - **Email:** Amazon SES with bounce and complaint handling.
   - **SMS:** Amazon SNS with delivery receipts.
   - **Push:** Firebase Cloud Messaging (FCM) with token management.
   - **WhatsApp:** Twilio API with template pre-approval (required by WhatsApp Business).
5. **Delivery Tracking** -- `DeliveryTracker` records the delivery attempt in PostgreSQL and monitors for delivery callbacks (SES notifications, SNS receipts, FCM responses). Failed deliveries are retried with exponential backoff (initial 30s, max 4 hours, up to 5 attempts).

## Integrations

| System | Direction | Protocol | Purpose |
|---|---|---|---|
| All domain services | Inbound | Kafka events | 25+ event types trigger notifications |
| notification-template-service | Outbound | REST API | Template rendering |
| Amazon SES | Outbound | AWS SDK | Email delivery |
| Amazon SNS | Outbound | AWS SDK | SMS delivery |
| Firebase FCM | Outbound | REST API | Push notification delivery |
| Twilio | Outbound | REST API | WhatsApp message delivery |
| PostgreSQL (notification_db) | Internal | JDBC | Notifications, preferences, delivery log |

## Architecture Decision Records

### ADR-049-1: Multi-Channel Delivery in a Single Service

**Status:** Accepted
**Context:** An early design proposed separate services per channel (email-service, sms-service, etc.). This would have simplified each service but fragmented delivery tracking, made channel preference logic distributed, and complicated the retry/fallback strategy where a failed SMS should fall back to email.
**Decision:** Consolidate all notification channels into a single `notification-service`. Channel senders are implemented as strategy plugins behind a common `ChannelSender` interface. The `NotificationRouter` orchestrates channel selection, rendering, dispatch, and fallback in one transactional flow.
**Consequences:** Unified delivery tracking and reporting. Cross-channel fallback is straightforward (e.g., push failure triggers email). The service is larger and more complex than a single-channel service, requiring careful separation of concerns internally. Channel-specific outages (e.g., Twilio downtime) are isolated by the plugin architecture and do not affect other channels.

### ADR-049-2: SQS/Kafka Consumer Over Webhook-Based Triggers

**Status:** Accepted
**Context:** Some notification platforms use webhook-based triggers where the originating service calls the notification service directly via REST. This couples the sender to the notification service's availability and introduces back-pressure concerns during high-volume events (e.g., batch payment reminders).
**Decision:** Use Kafka as the sole inbound trigger mechanism. Domain services publish events to their own Kafka topics without awareness of notifications. The notification-service subscribes to relevant topics and independently decides which notifications to send based on event-to-template mapping configuration.
**Consequences:** Full decoupling between domain services and notification delivery. The notification-service can be scaled independently and process events at its own pace using Kafka consumer group lag management. The trade-off is eventual consistency: notifications are not guaranteed to be sent in real time, but typical latency is under 5 seconds. Kafka consumer lag is monitored and alerted on.
