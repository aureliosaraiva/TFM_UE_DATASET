# Notification Template Service

**svc-50** // Notifications domain // team-notifications
Kotlin + Spring Boot // PostgreSQL (`notification_templates_db`)

---

## Overview

Every customer-facing message sent by CredityFlow -- emails, SMS texts, push notifications, WhatsApp messages -- starts as a template in this service. The Notification Template Service manages the full lifecycle of message templates: creation, versioning, variable validation, and on-demand rendering.

Templates use a Mustache-based syntax with custom extensions for locale-aware formatting (currency, dates, pluralization). Each template is associated with a channel (email, SMS, push, WhatsApp) and can have multiple versions, with one marked as active at any time.

The primary consumer of this service is `notification-service` (svc-49), which calls the render endpoint during message dispatch. Back-office users manage templates through the ops portal via `backoffice-bff`.

## Getting Started

```bash
# prerequisites: JDK 17+, PostgreSQL, Gradle

docker compose up -d postgres

# start the service (seeds example templates in local profile)
./gradlew bootRun -Dspring.profiles.active=local

# port: 8050

# tests
./gradlew test
```

The local seed data includes templates for common notification types (proposal_approved, payment_reminder, document_request) across all channels, so you can immediately test rendering.

## Architecture

This is a straightforward CRUD + rendering service with no event-driven components.

```
  backoffice-bff                notification-service
  (template mgmt)              (render at dispatch time)
       |                              |
       v                              v
  +---------+--------+    +-----------+-----------+
  | Template CRUD    |    | Template Render       |
  | (GET, PUT, POST) |    | (POST .../render)     |
  +---------+--------+    +-----------+-----------+
            |                         |
            v                         v
       +----+-------------------------+----+
       |        PostgreSQL                 |
       |   notification_templates_db       |
       |                                   |
       |   tables:                         |
       |   - templates (id, channel, name) |
       |   - template_versions (body, vars)|
       |   - template_metadata             |
       +-----------------------------------+
```

### Template Structure

A template record consists of:

- **id** -- stable identifier used by notification-service (e.g., `payment_reminder_sms`)
- **channel** -- `EMAIL`, `SMS`, `PUSH`, `WHATSAPP`
- **name** -- human-readable label
- **body** -- the template content with `{{variable}}` placeholders
- **subject** -- (email only) subject line template
- **required_variables** -- JSON array declaring what variables must be provided at render time
- **active_version** -- pointer to the current version

For email templates, the body supports HTML with inline CSS. SMS and WhatsApp templates are plain text with character limits enforced at the schema level.

## API Reference

### Get Template

```http
GET /api/v1/templates/{id}
```

Returns the template metadata and active version content.

### Update Template

```http
PUT /api/v1/templates/{id}
Content-Type: application/json

{
  "body": "Ola {{customer_name}}, sua proposta {{proposal_id}} foi {{status}}.",
  "subject": null,
  "required_variables": ["customer_name", "proposal_id", "status"]
}
```

Creates a new version of the template. The previous version is retained for audit purposes. The new version becomes active immediately.

### Render Template

```http
POST /api/v1/templates/{id}/render
Content-Type: application/json

{
  "variables": {
    "customer_name": "Maria Silva",
    "proposal_id": "PROP-2026-0412",
    "status": "aprovada"
  }
}
```

Returns the rendered output:

```json
{
  "rendered_body": "Ola Maria Silva, sua proposta PROP-2026-0412 foi aprovada.",
  "rendered_subject": null,
  "channel": "SMS",
  "character_count": 62
}
```

If required variables are missing, returns `400` with a list of the missing keys.

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `TEMPLATES_DB_URL` | JDBC URL | `jdbc:postgresql://localhost:5432/notification_templates_db` |
| `TEMPLATES_DB_USERNAME` | DB user | `templates` |
| `TEMPLATES_DB_PASSWORD` | DB password | -- |
| `SMS_MAX_CHARS` | Character limit for SMS templates | `160` |
| `WHATSAPP_MAX_CHARS` | Character limit for WhatsApp templates | `1024` |
| `PUSH_MAX_CHARS` | Character limit for push notification body | `256` |
| `TEMPLATE_CACHE_ENABLED` | Cache active template versions in memory | `true` |
| `TEMPLATE_CACHE_TTL_SECONDS` | In-memory cache TTL | `300` |

## Deployment

AWS EKS, `notifications` namespace, alongside notification-service.

```bash
helm upgrade --install notification-template-service \
  infra/helm/notification-template-service \
  --namespace notifications -f values-production.yaml
```

This is a low-traffic service (called only during notification dispatch and template editing):
- 2 replicas
- 256Mi-512Mi memory
- 200m-400m CPU

Database migrations via Flyway. Template data is critical -- ensure PostgreSQL backups are configured and tested.

## Monitoring

**Health:** `GET /actuator/health`

**Metrics:**
- `templates.render_total` -- counter by template ID and channel
- `templates.render_latency_ms` -- histogram
- `templates.render_errors_total` -- counter (missing variables, invalid templates)
- `templates.cache_hit_ratio` -- gauge

**Dashboard:** Grafana > Notifications > Template Service

**Alerts:**
- Render error rate > 1% (usually indicates a template was updated with mismatched variables)
- Render latency p99 > 200ms

## Known Issues

1. **No template preview/sandbox.** Editors must save a template and trigger a test notification to see the rendered result. A preview endpoint that renders without persisting is a frequently requested feature.

2. **No approval workflow.** Template changes go live immediately. For regulated communications (e.g., collection notices with legal language), there should be a review-and-approve step. Currently this is handled out-of-band via Slack sign-off, which is error-prone.

3. **HTML email templates are fragile.** Email body templates contain inline CSS for compatibility with email clients. There is no visual editor or preview -- editors work with raw HTML in a text area. Template validation only checks variable syntax, not HTML correctness, so broken layouts have shipped to production.
