[← Table of Contents](../../README.md)

## Core Domain Model

This section defines the core entities, relationships, and abstractions underlying the notification system.  
The domain model ensures consistency across routing, personalization, delivery tracking, provider interactions, and auditing.

### Overview

The notification platform is modeled around the concept of a **Notification Request** that expands into one or more **Channel Attempts**, each with its own lifecycle and delivery semantics. Templates, preferences, and provider configurations influence how these attempts are created, rendered, and executed.

The domain model is intentionally channel-agnostic and supports extensibility for new channels, tenants, and provider types.

---

### Core Domain Concepts

#### **NotificationRequest**
Represents the initial request from a producer service.
- Unique request ID / idempotency key  
- Notification type  
- User identifier  
- Payload for template rendering  
- Priority (e.g., high, bulk)  
- Metadata (tenant, correlation IDs)

This object is immutable once accepted.

---

#### **NotificationType**
Defines a category or intent of notification.
- Name (e.g., `high_usage_alert`, `billing_summary`)  
- Allowed channels  
- Channel priority & fallback order  
- Default template mappings per channel  
- Regulatory flags (e.g., must send via email or paper)

Notification types allow consistent behavior across producers.

---

#### **NotificationTemplate**
Defines channel-specific versions of the message.
- Channel (email, SMS, push, paper, etc.)  
- Locale / language  
- Version ID  
- Template body (HTML, text, JSON, etc.)  
- Static variables and supported dynamic fields

Templates must be deterministic and auditable.

---

#### **User**
Represents the target recipient.
- Unique user ID  
- Contact attributes (email, phone, address)  
- Locale, timezone  
- Optional tenant-specific metadata

User data is not directly stored in notifications; only references are used.

---

#### **UserPreference**
Defines the user’s delivery preferences per notification type.
- Preferred channels  
- Priority ordering  
- Opt-in / opt-out settings  
- Regulatory constraints  
- Do-not-disturb windows  
- Tenant-level overrides

Preferences heavily influence routing decisions.

---

#### **ChannelAttempt**
Represents an attempt to deliver a notification through a specific channel.
- Attempt ID  
- Channel (email, SMS, push, paper)  
- Resolved provider configuration  
- Payload after template rendering  
- Attempt status (queued, sent, delivered, failed, retried, etc.)  
- Retry count  
- Timestamps for all state transitions

A single NotificationRequest may expand into multiple ChannelAttempts.

---

#### **ProviderConfig**
Defines configuration and credentials for a channel provider.
- Channel (email, SMS, etc.)  
- Provider name (SendGrid, Twilio, etc.)  
- API credentials (securely stored)  
- Rate limits and quotas  
- Connection parameters  
- Provider-specific global settings

This enables the system to dynamically select or failover providers.

---

#### **NotificationEvent / AuditLog**
Immutable events emitted through the lifecycle.
- Request received  
- Routing resolved  
- Template rendered  
- Channel attempt created  
- Provider response received  
- Delivery status change  
- Retry/fallback decision  

These events form a full audit trail and power observability.

---

### Diagram: Domain Model / ER

![Domain Model ER](./diagrams/domain-model-er.svg)

**Recommended relationships to include in the diagram:**

- **User 1—1..n UserPreference**  
  Each user may define preferences per notification type.

- **NotificationType 1—1..n NotificationTemplate**  
  Each type has channel and locale-specific templates.

- **NotificationRequest 1—1..n ChannelAttempt**  
  A request expands into multiple channel deliveries.

- **ChannelAttempt 1—1 ProviderConfig**  
  The system resolves a provider before sending.

- **NotificationRequest 1—n NotificationEvent**  
  Every state transition and lifecycle update is recorded.

- **ChannelAttempt 1—n NotificationEvent**  
  Each attempt generates multiple events (queued, sent, delivered…).

---

### Modeling Rationale

- **Separation of Request vs. Attempt**  
  Ensures routing logic can generate multiple attempts with independent states.

- **Templates & Preferences decoupled from requests**  
  Allows updates over time without modifying historical data.

- **ProviderConfig as a separate entity**  
  Enables multi-provider routing, health-based selection, and dynamic failover.

- **Event-sourced / audit-driven approach**  
  Simplifies debugging, ensures compliance, and supports operational replay.

- **Entities designed to be channel-agnostic**  
  Ensures new channels require minimal modifications.

---
[← Table of Contents](../../README.md)