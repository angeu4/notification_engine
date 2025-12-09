[← Table of Contents](../../README.md)

[← Previous page: Appendix A: Sequence Diagrams](a_sequence_diagrams.md)

## Appendix B: Configuration Examples

This appendix provides configuration samples used by different components of the notification platform.  
These examples demonstrate how routing, templates, providers, preferences, and system-level behaviors can be defined using declarative configuration.

---

## B1. Notification Type Configuration

Defines the intended behavior, allowed channels, priority order, and fallback logic for each notification type.

```yaml
notificationTypes:
  - id: high_usage_alert
    description: "Alert sent when consumption exceeds threshold"
    allowedChannels:
      - sms
      - email
    channelPriority:
      - sms
      - email
    fallback:
      enabled: true
      order:
        - sms
        - email
    regulatory:
      requireSpecificChannel: false
    templateMapping:
      sms: high_usage_alert_sms_v3
      email: high_usage_alert_email_v5
```

---

## B2. Template Metadata Configuration

Defines templates by channel, locale, version, and tenant overrides.

```yaml
templates:
  - id: high_usage_alert_email_v5
    channel: email
    locale: en-US
    version: 5
    tenant: default
    subject: "Your energy usage is higher than usual"
    bodyPath: "/templates/email/high_usage_alert_v5.html"

  - id: high_usage_alert_sms_v3
    channel: sms
    locale: en-US
    version: 3
    tenant: default
    text: "Your usage is high. You're at {{usage}} vs {{threshold}}."
```

---

## B3. User Preference Configuration

Defines routing preferences, DND windows, and opt-in status.

```yaml
userPreferences:
  userId: "12345"
  preferences:
    high_usage_alert:
      preferredChannels:
        - sms
        - email
      optOut: false
      dnd:
        enabled: true
        start: "22:00"
        end: "07:00"
```

---

## B4. Provider Configuration

Defines providers and credentials for each channel.

```yaml
providers:
  sms:
    - name: twilio_primary
      endpoint: "https://api.twilio.com/send"
      apiKeySecretRef: "secrets/twilio/apiKey"
      maxRps: 1000
      fallbackPriority: 1

    - name: nexmo_backup
      endpoint: "https://api.nexmo.com/sms"
      apiKeySecretRef: "secrets/nexmo/apiKey"
      maxRps: 500
      fallbackPriority: 2

  email:
    - name: sendgrid_primary
      endpoint: "https://api.sendgrid.com/v3/mail/send"
      apiKeySecretRef: "secrets/sendgrid/key"
      maxRps: 2000
      fallbackPriority: 1
```

---

## B5. Routing Rules Configuration

Defines tenant-level routing overrides, fallback behavior, and throttling.

```yaml
routingRules:
  tenantId: "tenantA"
  rules:
    high_usage_alert:
      allowedChannels:
        - sms
        - email
      channelPriority:
        - email
        - sms
      fallbackEnabled: true
    billing_statement:
      allowedChannels:
        - paper
        - email
      fallbackEnabled: false
```

---

## B6. Throttling & Rate Limit Configuration

Defines rate limits globally, per tenant, and per provider.

```yaml
rateLimits:
  global:
    maxRequestsPerSecond: 5000

  perTenant:
    tenantA:
      maxRequestsPerSecond: 2000
    tenantB:
      maxRequestsPerSecond: 500

  providers:
    twilio_primary:
      maxRps: 1000
    sendgrid_primary:
      maxRps: 2000
```

---

## B7. Worker Configuration

Defines worker pools per channel, concurrency settings, and retry strategies.

```yaml
workers:
  sms:
    concurrency: 50
    retryPolicy:
      maxAttempts: 5
      backoff: exponential
      initialDelayMs: 500
      maxDelayMs: 10000

  email:
    concurrency: 200
    retryPolicy:
      maxAttempts: 3
      backoff: linear
      delayMs: 2000

  paper:
    concurrency: 5
    batchSize: 1000
```

---

## B8. DLQ & Retry Configuration

Defines DLQ routing, retention, and retry behavior.

```yaml
dlq:
  retentionDays: 14
  notificationTypes:
    - high_usage_alert
    - billing_statement

retryQueues:
  sms_retry:
    backoffStrategy: exponential
    maxAttempts: 5
  email_retry:
    backoffStrategy: exponential
    maxAttempts: 3
```

---

## B9. Feature Flags

Feature flags allow safe rollout of new channels, providers, or routing rules.

```yaml
featureFlags:
  enablePushNotifications: true
  enableDynamicProviderRouting: false
  useNewTemplateEngine: false
  enableA2P10DLCComplianceCheck: true
```

---
[→ Next page: Appendix C: Implementation Sketch](c_implementation_sketch.md)

[← Table of Contents](../../README.md)