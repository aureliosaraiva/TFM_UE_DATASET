# Architecture: notification-template-service

## Component Diagram

```
+----------------------------+
| notification-service       |
+-------------+--------------+
              |
     REST (render, preview)
              |
              v
+------------------------------------------------------------------------+
|               notification-template-service                            |
|                                                                        |
|  +---------------------+    +------------------------+                 |
|  | TemplateController  |--->| TemplateService        |                 |
|  | (REST API)          |    +---+--------+-----------+                 |
|  +---------------------+        |        |                             |
|                                 v        v                             |
|        +-------------------+  +------------------------+               |
|        | RenderEngine      |  | VersionManager         |               |
|        | (Mustache/        |  | (template versioning)  |               |
|        |  Handlebars)      |  +------------------------+               |
|        +-------------------+                                           |
|                                                                        |
|                          +------------------+                          |
|                          | PostgreSQL       |                          |
|                          | notification_    |                          |
|                          | templates_db     |                          |
|                          +------------------+                          |
+------------------------------------------------------------------------+
              |
     REST (CRUD, preview)
              |
              v
+----------------------------+
| backoffice-bff             |
+----------------------------+
```

## Data Flow

1. **Template Management** -- The backoffice team creates and edits notification templates via the backoffice-bff, which calls the `TemplateController` CRUD endpoints. Each template has a unique key (e.g., `credit_approved_email`), a channel type, a subject pattern, and a body pattern using Mustache/Handlebars syntax.
2. **Version Control** -- Every template edit creates a new version. `VersionManager` maintains the version history and tracks which version is currently active. Previous versions can be restored via the API. Templates are never deleted, only deactivated.
3. **Render Request** -- When `notification-service` needs to send a notification, it calls `POST /api/v1/templates/{key}/render` with the template variables (customer name, amounts, dates, etc.). The `RenderEngine` loads the active version of the requested template, applies the variables using Mustache/Handlebars, and returns the rendered subject and body.
4. **Preview** -- The backoffice team can call `POST /api/v1/templates/{key}/preview` with sample data to see how a template renders before activating it. This supports iterative editing without affecting production notifications.

## Integrations

| System | Direction | Protocol | Purpose |
|---|---|---|---|
| notification-service | Inbound | REST API | Template rendering for notifications |
| backoffice-bff | Inbound | REST API | Template CRUD and preview |
| PostgreSQL (notification_templates_db) | Internal | JDBC | Template storage and version history |

## Architecture Decision Records

### ADR-050-1: Separate Template Service

**Status:** Accepted
**Context:** Initially, template management was embedded within the notification-service. This coupling meant that template edits required redeployment of the notification service, and the backoffice team needed engineering support for every copy change. Template rendering logic also added complexity to an already large service responsible for multi-channel delivery.
**Decision:** Extract template management into a dedicated `notification-template-service` with its own database and REST API. The notification-service calls this service at render time. The backoffice-bff integrates directly with the template service for CRUD operations, enabling non-technical team members to manage notification copy through the backoffice UI.
**Consequences:** The backoffice team can create, edit, preview, and activate templates independently without engineering involvement or deployments. Template versioning provides an audit trail and rollback capability. The trade-off is an additional REST call during notification dispatch, adding ~5-10ms latency. Templates are cached in the notification-service for frequently used keys to mitigate this. The template service itself is simple (no events, no async flows) and rarely changes, making it low-maintenance.
