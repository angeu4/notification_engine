[← Table of Contents](../../README.md)

[← Previous page: Appendix C: Implementation Sketch](c_implementation_sketch.md)

## Appendix D: API Reference

This appendix defines a complete, production-grade API specification for the Notification Platform.  
It includes all endpoints required for notification submission, idempotency, template management, preferences, configuration introspection, and provider callbacks.

The goal is to demonstrate how a real-world notification API would be formally documented.

---

## D1. Authentication & Versioning

### **Authentication**
All endpoints require one of the following:
- **API Key** (service-to-service)
- **OAuth2 Client Credentials** (recommended for multi-tenant)
- **mTLS** (optional for high-security deployments)

Example header:
```
Authorization: Bearer <token>
X-Tenant-ID: tenantA
```

---

### **API Versioning Strategy**
- URL versioning: `/v1/...`
- Backward-compatible changes allowed.
- Breaking changes always require a new version.

---

## D2. Core Notification Submission API

## **POST /v1/notifications**

Submit a new notification request.

### Request Body
```json
{
  "notificationType": "high_usage_alert",
  "userId": "12345",
  "payload": {
    "usage": 502,
    "threshold": 450
  },
  "priority": "high",
  "locale": "en-US",
  "idempotencyKey": "req-12345-xyz",
  "metadata": {
    "traceId": "abc-def",
    "tenantContext": "premium"
  }
}
```

### Response
```json
{
  "status": "accepted",
  "requestId": "req_88990011"
}
```

### Error Examples
```json
{
  "errorCode": "INVALID_PAYLOAD",
  "message": "Missing required field: userId"
}
```

---

## D3. Get Notification Status

## **GET /v1/notifications/{requestId}**

Retrieve the status of a full notification (including all channel attempts).

### Response
```json
{
  "requestId": "req_88990011",
  "type": "high_usage_alert",
  "userId": "12345",
  "status": "delivered",
  "channels": [
    {
      "channel": "sms",
      "attemptId": "att_001",
      "status": "delivered",
      "provider": "twilio_primary"
    }
  ]
}
```

## D4. Channel Attempt Status

## **GET /v1/attempts/{attemptId}**

Retrieve detailed per-channel delivery lifecycle.

### Sample Response
```json
{
  "attemptId": "att_001",
  "channel": "sms",
  "status": "delivered",
  "provider": "twilio_primary",
  "retries": 1,
  "events": [
    { "state": "QUEUED", "timestamp": "..." },
    { "state": "SENT", "timestamp": "..." },
    { "state": "DELIVERED", "timestamp": "..." }
  ]
}
```

---

## D5. Template Management API

## **GET /v1/templates**

List all templates for a tenant.

## **POST /v1/templates**

Create a new template version.

### Request
```json
{
  "templateId": "billing_summary_email",
  "channel": "email",
  "locale": "en-US",
  "version": 3,
  "subject": "Your Monthly Billing Summary",
  "body": "<html>...{{amount}}...</html>"
}
```

---

## D6. Template Rendering API (Preview)

## **POST /v1/templates/preview**

Allows UI tools or QA to render a template without sending a notification.

```json
{
  "templateId": "billing_summary_email",
  "locale": "en-US",
  "data": { "amount": "142.77" }
}
```

---

## D7. User Preferences API

## **GET /v1/users/{userId}/preferences**

## **PATCH /v1/users/{userId}/preferences**

```json
{
  "high_usage_alert": {
    "preferredChannels": ["sms", "email"],
    "optOut": false,
    "dnd": { "enabled": true, "start": "22:00", "end": "07:00" }
  }
}
```

---

## D8. Routing Rules API (Tenant-Level)

```json
{
  "high_usage_alert": {
    "allowedChannels": ["sms", "email"],
    "channelPriority": ["sms", "email"],
    "fallbackEnabled": true
  }
}
```

---

## D9. Provider Callback Webhooks

## **POST /v1/providers/callbacks**

Providers call this endpoint asynchronously.

### Example Callback Payload
```json
{
  "provider": "twilio",
  "attemptId": "att_001",
  "status": "delivered",
  "timestamp": "2024-01-12T10:22:33Z"
}
```

---

## D10. Error Codes

```json
{
  "INVALID_PAYLOAD": "Malformed body or missing fields",
  "UNAUTHORIZED": "Invalid API key or token",
  "NOT_FOUND": "Notification or attempt not found",
  "TEMPLATE_ERROR": "Template rendering failure",
  "RATE_LIMIT": "Rate limit exceeded",
  "INTERNAL_ERROR": "Unexpected system failure"
}
```

---

## D11. Rate Limiting

- Per-tenant rate limits  
- Global rate limits  
- Per-channel rate limits  
- Provider rate limits  

Rate-limit headers:
```
X-RateLimit-Limit: 2000
X-RateLimit-Remaining: 1875
X-RateLimit-Reset: 1700000034
```

---
[← Table of Contents](../../README.md)