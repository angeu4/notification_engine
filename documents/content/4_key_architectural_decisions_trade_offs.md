[← Table of Contents](../../README.md)

## Key Architectural Decisions & Trade-offs

This section outlines the major architectural decisions that shape the design of the notification service.  
Each decision includes: **Context**, **Options Considered**, **Decision**, **Rationale**, and **Implications**.

---

### **D1: Event-Driven Asynchronous Pipeline vs. Synchronous Delivery**

**Context**  
Notifications may require complex routing, personalization, template rendering, retries, and provider failover. Producer systems should not block on downstream delivery delays.

**Options Considered**  
1. **Synchronous delivery:** Producer waits until delivery attempt is complete.  
2. **Asynchronous event-driven pipeline:** Producer receives immediate acknowledgment; the system processes delivery independently.

**Decision**  
Use an **asynchronous, event-driven pipeline** backed by queues/streams.

**Rationale**
- Decouples producer performance from channel provider latency.  
- Enables high throughput and burst absorption.  
- Natural fit for retry, fallback, and long-running workflows (e.g., paper mail).  
- Avoids synchronous bottlenecks and cascading failures.

**Implications**
- Delivery confirmation becomes eventual, not immediate.  
- Requires persistent audit and idempotency handling.  
- Improves scalability and resilience significantly.

---

### **D2: Channel-Agnostic Core with Pluggable Channel Adapters**

**Context**  
Producers should not need to know or care about which channels are used or how providers work.

**Options Considered**  
1. Hardcoded channel logic inside the core service.  
2. Pluggable adapter model for each channel.

**Decision**  
Adopt a **channel-agnostic core** where routing picks channels, and each channel is handled by a **pluggable adapter or microservice**.

**Rationale**
- Clean separation of concerns.  
- Easy onboarding of new channels (push, WhatsApp, voice, etc.).  
- Allows independent scaling per channel.  
- Improves maintainability and isolates provider failures.

**Implications**
- Requires common delivery interface abstractions.  
- Template formats may differ per channel, requiring careful design.

---

### **D3: Queueing Strategy (Per-Channel / Per-Priority / Per-Tenant)**

**Context**  
Millions of notifications per day require isolation between channels, tenants, and workload types.

**Options Considered**  
1. Single global queue.  
2. Per-channel queues.  
3. Per-channel + per-priority queues.  
4. Per-tenant queues.

**Decision**  
Use **per-channel queues**, optionally segmented by **priority**.

**Rationale**
- Isolates issues (e.g., SMS provider outage doesn't impact email).  
- Allows channel-specific autoscaling and throttling.  
- Priority queues ensure urgent alerts are handled ahead of bulk messages.

**Implications**
- Slightly more operational overhead managing multiple queues.  
- Requires routing logic to publish tasks to the right queue.  
- Supports horizontal scale and fault isolation.

---

### **D4: Rules Engine vs. Configuration-Driven Routing Logic**

**Context**  
Routing involves user preferences, fallback rules, tenant overrides, blackout windows, and regulatory requirements.

**Options Considered**  
1. Hardcoded routing logic.  
2. Config-driven JSON/YAML-based routing rules.  
3. Full-blown rules engine.

**Decision**  
Start with **configuration-driven routing rules**, structured and validated.  
Evolve to a rules engine only when complexity demands it.

**Rationale**
- Configs remain readable and auditable by operational teams.  
- Lower complexity and operational footprint than a rules engine.  
- Supports most routing scenarios through structured configs.

**Implications**
- Complex conditional logic may eventually outgrow configs.  
- Need tooling for validating and deploying routing configs safely.

---

### **D5: Delivery Semantics — At-Least-Once Delivery with Idempotency**

**Context**  
External providers may time out, return ambiguous responses, or drop connections. Retries are inevitable.

**Options Considered**  
1. At-most-once delivery (no retries).  
2. At-least-once delivery + idempotency keys.  
3. Exactly-once delivery (hard across distributed boundaries).

**Decision**  
Use **at-least-once delivery with strict idempotency controls**.

**Rationale**
- Prevents message loss under failures.  
- Feasible across distributed systems without overly complex protocols.  
- Idempotency eliminates duplicate user-visible notifications.

**Implications**
- Requires a short-lived deduplication store.  
- Providers may still send duplicates; the system must normalize them.  
- State machines must be idempotent.

---

### **D6: Data Storage Choices (Relational vs. NoSQL vs. Time-Series)**

**Context**  
Different data domains exhibit different access patterns:
- Templates and preferences need structured, queryable storage.
- Notification events are append-only and high-volume.
- Channel attempts need fast writes and time-based retrievals.

**Options Considered**  
1. Single relational DB.  
2. NoSQL for preferences + relational for templates.  
3. Time-series or log store for audit/tracking.  
4. Combination of specialized stores.

**Decision**  
Use **multiple purpose-fit stores**:
- Relational DB for templates, provider configs.  
- NoSQL (or distributed KV store) for user preferences.  
- Time-series/log store for notification events and audit logs.  
- Cache (Redis-like) for high-read elements.

**Rationale**
- Matches data models to access patterns.  
- Improves latency and scalability.  
- Reduces operational load on any single store.

**Implications**
- Requires clear data retention policies.  
- More systems to manage but significantly improved performance.

---

### **D7: Multi-Provider Strategy — Active-Active vs. Active-Passive**

**Context**  
Providers (email/SMS/push vendors) can experience outages, throttling, or degraded performance.

**Options Considered**  
1. Active-passive failover.  
2. Active-active dynamic routing based on health and cost.

**Decision**  
Adopt **active-active** provider selection where feasible, with auto-failover.

**Rationale**
- Improves reliability and resilience.  
- Reduces outage blast radius.  
- Enables cost-aware routing (e.g., use lower-cost provider when healthy).

**Implications**
- Requires provider health scoring and telemetry.  
- Increased operational complexity.  
- Routing logic must be dynamic and observability-driven.

---

### **D8: Microservices vs. Modular Monolith**

**Context**  
Should channels, routing, templates, and preferences live in one service or multiple?

**Options Considered**  
1. Modular monolith.  
2. Separate microservices for routing, templates, preferences, channels.

**Decision**  
Use a **microservice-style decomposition**, but keep boundaries **cohesive**:
- Routing/orchestration as a core service.
- Channel services independently scalable.
- Template and preference services modular but lightweight.

**Rationale**
- Enables independent scaling and deployments.  
- Reduces blast radius across components.  
- Allows teams to evolve channels independently.

**Implications**
- Requires strong observability and contract stability.  
- Slightly higher operational cost vs. monolith.

---

### **D9: Preference Reads — Cached vs. Strong Consistency**

**Context**  
Notification routing requires low-latency preference reads.

**Options Considered**  
1. Always read from a strongly consistent DB.  
2. Cache preferences with eventual consistency.  
3. Push-based caching (event-driven updates).

**Decision**  
Use **cache-backed preference reads** with event-driven updates.

**Rationale**
- Lowers read latency for routing.  
- Reduces database load.  
- Event-driven updates ensure consistency without polling.

**Implications**
- Potential for brief inconsistency windows.  
- Must validate cache invalidation and TTL strategy.

---

### **D10: Template Rendering — Real-Time vs. Pre-Rendered Fragments**

**Context**  
Rendering templates on demand may be expensive at scale.

**Options Considered**  
1. Real-time rendering only.  
2. Pre-render static fragments + dynamic merge.  
3. Fully pre-rendered templates.

**Decision**  
Use **hybrid rendering**:
- Frequently changing dynamic sections rendered in real-time.  
- Static sections cached or pre-rendered per tenant/language.

**Rationale**
- Reduces overall rendering cost.  
- Ensures personalization accuracy.  
- Improves performance at high volume.

**Implications**
- Requires template cache invalidation strategy.  
- Additional tooling needed for preview/rollback.


---
[← Table of Contents](../../README.md)