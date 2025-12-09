[← Table of Contents](../../README.md)

[← Previous page: Appendix B: Configuration Examples](b_configuration_examples.md)

## Appendix C: Implementation Sketch

This appendix provides a lightweight implementation-oriented sketch illustrating how key components of the notification platform may be structured in code.  
The sketches below are language-agnostic but use a pseudocode style similar to Python/TypeScript for readability.

They are not meant to be production-ready; rather, they show how the architecture maps cleanly to modular, testable components.

---

## C1. Notification API Handler (Ingress Layer)

```python
class NotificationAPI:
    def __init__(self, orchestrator, idempotency_store):
        self.orchestrator = orchestrator
        self.idempotency_store = idempotency_store

    def create_notification(self, request):
        key = request.idempotency_key

        # Idempotency check
        if self.idempotency_store.exists(key):
            return self.idempotency_store.get_response(key)

        # Basic validation
        validate(request)

        # Normalize and publish event
        event = NotificationRequestEvent.from_request(request)
        self.orchestrator.publish_request(event)

        # Cache response for idempotency guarantee
        response = {"status": "accepted", "requestId": event.request_id}
        self.idempotency_store.save(key, response)

        return response
```

---

## C2. Orchestrator & Routing Engine

```python
class Orchestrator:
    def __init__(self, preference_store, routing_rules, template_service, channel_queues):
        self.preference_store = preference_store
        self.routing_rules = routing_rules
        self.template_service = template_service
        self.channel_queues = channel_queues

    def handle_notification_request(self, event):
        prefs = self.preference_store.get_user_preferences(event.user_id)
        rules = self.routing_rules.get(event.notification_type)

        channels = resolve_channels(prefs, rules)

        for channel in channels:
            attempt = ChannelAttempt(
                request_id=event.request_id,
                channel=channel,
                payload=event.payload,
            )

            # Render content ahead of queueing
            attempt.rendered = self.template_service.render(
                notification_type=event.notification_type,
                channel=channel,
                locale=event.locale,
                data=event.payload
            )

            # Queue attempt
            self.channel_queues[channel].push(attempt)
```

---

## C3. Channel Worker (Delivery Layer)

```python
class ChannelWorker:
    def __init__(self, provider_router, retry_scheduler, state_store):
        self.provider_router = provider_router
        self.retry_scheduler = retry_scheduler
        self.state_store = state_store

    def process_attempt(self, attempt):
        provider = self.provider_router.select_provider(attempt.channel)

        try:
            response = provider.send(attempt.rendered)
            if response.success:
                self.state_store.update(attempt.id, "SENT")
            else:
                self.handle_failure(attempt, response.error)
        except Exception as exc:
            self.handle_failure(attempt, str(exc))

    def handle_failure(self, attempt, error):
        if is_retryable(error):
            self.retry_scheduler.schedule(attempt)
            self.state_store.update(attempt.id, "RETRY_PENDING")
        else:
            self.state_store.update(attempt.id, "FAILED")
            emit_fallback_if_allowed(attempt)
```

---

## C4. Provider Router (Active-Active or Failover Logic)

```python
class ProviderRouter:
    def __init__(self, provider_configs, health_monitor):
        self.providers = load_providers(provider_configs)
        self.health_monitor = health_monitor

    def select_provider(self, channel):
        healthy = [
            p for p in self.providers[channel]
            if self.health_monitor.is_healthy(p.name)
        ]

        # Fallback to priority-based ranking
        healthy.sort(key=lambda p: p.fallback_priority)

        if not healthy:
            raise Exception("No healthy providers available")

        return healthy[0]
```

---

## C5. Template Rendering Engine

```python
class TemplateService:
    def __init__(self, template_repo, cache):
        self.template_repo = template_repo
        self.cache = cache

    def render(self, notification_type, channel, locale, data):
        template_id = self.template_repo.resolve(
            notification_type, channel, locale
        )

        template = self.cache.get(template_id)
        if not template:
            template = self.template_repo.load(template_id)
            self.cache.set(template_id, template)

        return template.render(data)
```

---

## C6. Provider Adapter Interface

```python
class ProviderAdapter:
    def send(self, formatted_payload):
        raise NotImplementedError()
```

### Example: Email Provider Adapter

```python
class SendGridAdapter(ProviderAdapter):
    def __init__(self, api_key, endpoint):
        self.api_key = api_key
        self.endpoint = endpoint

    def send(self, payload):
        resp = http_post(self.endpoint, payload, headers={
            "Authorization": f"Bearer {self.api_key}"
        })
        return ProviderResponse(success=resp.status == 202)
```

---

## C7. Retry Scheduler

```python
class RetryScheduler:
    def __init__(self, retry_queues):
        self.retry_queues = retry_queues

    def schedule(self, attempt):
        attempt.retry_count += 1
        delay = compute_backoff(attempt.retry_count)
        self.retry_queues[attempt.channel].delay_push(attempt, delay)
```

---

## C8. Health Monitor (Provider Health Score)

```python
class ProviderHealthMonitor:
    def __init__(self):
        self.health = {}

    def update(self, provider_name, status):
        self.health[provider_name] = status

    def is_healthy(self, provider_name):
        return self.health.get(provider_name, True)
```

---

## C9. Preference Store (Low-Latency KV Model)

```python
class PreferenceStore:
    def __init__(self, kv):
        self.kv = kv

    def get_user_preferences(self, user_id):
        prefs = self.kv.get(f"user:{user_id}:prefs")
        return prefs or default_preferences()
```

---

## C10. Event Emission (Audit Trail)

```python
def emit_event(event_type, entity_id, metadata):
    log.info({
        "event": event_type,
        "entityId": entity_id,
        "metadata": metadata,
        "timestamp": now()
    })
```

---
[→ Next page: Appendix D: API Reference](d_api_reference.md)

[← Table of Contents](../../README.md)