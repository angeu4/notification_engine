[← Table of Contents](../../README.md)

## Appendix A: Sequence Diagrams

This appendix provides sequence diagrams illustrating the end-to-end flows of the notification platform.  
These diagrams cover the most important operational scenarios: request ingestion, routing, rendering, channel delivery, retries, fallback flows, and provider callbacks.

---

### A1. Notification Request → Delivery (Happy Path)

```mermaid
sequenceDiagram
    participant Producer
    participant API as Notification API
    participant Orchestrator
    participant PrefSvc as Preference Service
    participant TemplateSvc as Template Service
    participant Queue as Channel Queue
    participant Worker as Channel Worker
    participant Provider

    Producer->>API: POST /v1/notifications
    API->>API: Validate & generate idempotency key
    API->>Orchestrator: Publish NotificationRequest event
    Orchestrator->>PrefSvc: Fetch user preferences
    PrefSvc-->>Orchestrator: Preferences
    Orchestrator->>Orchestrator: Resolve eligible channels
    Orchestrator->>TemplateSvc: Fetch template metadata
    TemplateSvc-->>Orchestrator: Template info
    Orchestrator->>TemplateSvc: Render template
    TemplateSvc-->>Orchestrator: Rendered content
    Orchestrator->>Queue: Enqueue ChannelAttempt
    Worker->>Queue: Pull ChannelAttempt
    Worker->>Provider: Send request
    Provider-->>Worker: 200 OK / Delivery accepted
    Worker->>Orchestrator: Emit SENT event
    Provider-->>Worker: Delivery callback (DELIVERED)
    Worker->>Orchestrator: Emit DELIVERED event
```

---

### A2. Retry Flow (Transient Provider Failure)

```mermaid
sequenceDiagram
    participant Worker as Channel Worker
    participant Provider
    participant Orchestrator
    participant RetryQ as Retry Queue

    Worker->>Provider: Send notification
    Provider-->>Worker: 5xx / timeout / throttled
    Worker->>Worker: Evaluate retry policy (backoff + jitter)
    Worker->>RetryQ: Enqueue retry attempt
    RetryQ->>Worker: Deliver retry message after delay
    Worker->>Provider: Retry send
    Provider-->>Worker: Success
    Worker->>Orchestrator: Emit SENT event
```

---

### A3. Fallback Flow (Primary Channel Failure → Secondary Channel)

```mermaid
sequenceDiagram
    participant Orchestrator
    participant WorkerA as Primary Channel Worker
    participant WorkerB as Fallback Channel Worker
    participant ProviderA
    participant ProviderB

    WorkerA->>ProviderA: Send
    ProviderA-->>WorkerA: Hard failure (non-retryable)
    WorkerA->>Orchestrator: Emit FAILED_PRIMARY event
    Orchestrator->>Orchestrator: Check fallback rules
    Orchestrator->>WorkerB: Create fallback ChannelAttempt
    WorkerB->>ProviderB: Send fallback notification
    ProviderB-->>WorkerB: Success
    WorkerB->>Orchestrator: Emit FALLBACK_SENT event
```

---

### A4. Provider Callback Flow

```mermaid
sequenceDiagram
    participant Provider
    participant CallbackAPI as Callback/Webhook API
    participant Orchestrator
    participant Events as Event Store

    Provider->>CallbackAPI: POST delivery-status callback
    CallbackAPI->>CallbackAPI: Validate signature & payload
    CallbackAPI->>Orchestrator: Publish PROVIDER_CALLBACK event
    Orchestrator->>Events: Update ChannelAttempt state
    Orchestrator->>Events: Emit DELIVERED/BOUNCED/FAILED event
```

---

### A5. Dead-Letter Queue Handling

```mermaid
sequenceDiagram
    participant Worker
    participant DLQ as Dead-Letter Queue
    participant Ops as Ops Dashboard
    participant Orchestrator

    Worker->>Worker: Max retries exceeded
    Worker->>DLQ: Move message to DLQ
    Worker->>Orchestrator: Emit DLQ_PLACED event
    Ops->>DLQ: Inspect message via dashboard
    Ops->>Orchestrator: Replay command (optional)
    Orchestrator->>Worker: Reprocess DLQ entry
```

---

### A6. Preference-Based Suppression Flow

```mermaid
sequenceDiagram
    participant Producer
    participant API
    participant Orchestrator
    participant PrefSvc as Preference Service

    Producer->>API: POST /v1/notifications
    API->>Orchestrator: Publish NotificationRequest
    Orchestrator->>PrefSvc: Fetch preferences
    PrefSvc-->>Orchestrator: User opted-out of channel/type
    Orchestrator->>Orchestrator: Suppress notification
    Orchestrator->>Producer: Emit SUPPRESSED event (async)
```

---

### A7. Template Rendering Failure Flow

```mermaid
sequenceDiagram
    participant Orchestrator
    participant TemplateSvc as Template Service
    participant DLQ
    participant Events

    Orchestrator->>TemplateSvc: Render template
    TemplateSvc-->>Orchestrator: Rendering error (missing vars)
    Orchestrator->>Events: Emit TEMPLATE_ERROR event
    Orchestrator->>DLQ: Move notification to DLQ
    Orchestrator->>Events: Emit DLQ_PLACED event
```

---

### A8. Multi-Channel Expansion Flow (e.g., SMS + Email)

```mermaid
sequenceDiagram
    participant Orchestrator
    participant TemplateSvc
    participant QueueSMS as SMS Queue
    participant QueueEmail as Email Queue

    Orchestrator->>Orchestrator: Resolve multiple channels (SMS, Email)
    Orchestrator->>TemplateSvc: Render SMS template
    TemplateSvc-->>Orchestrator: SMS content
    Orchestrator->>QueueSMS: Enqueue SMS attempt
    Orchestrator->>TemplateSvc: Render Email template
    TemplateSvc-->>Orchestrator: Email content
    Orchestrator->>QueueEmail: Enqueue Email attempt
```

---

### A9. Multi-Tenant Routing Flow

```mermaid
sequenceDiagram
    participant Producer
    participant API
    participant Orchestrator
    participant TenantCfg as Tenant Routing Config
    participant Worker
    Producer->>API: Send notification (TenantID included)
    API->>Orchestrator: Publish NotificationRequest
    Orchestrator->>TenantCfg: Fetch tenant-specific routing rules
    TenantCfg-->>Orchestrator: Return rules
    Orchestrator->>Worker: Dispatch attempts per tenant config
    Worker->>Worker: Execute send with tenant-specific provider settings
```

---
[← Table of Contents](../../README.md)