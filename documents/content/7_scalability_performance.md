[← Table of Contents](../../README.md)

[← Previous page: Detailed System Design](6_detailed_system_design.md)

## Scalability & Performance

This section outlines how the notification platform scales to handle millions of notifications per day while maintaining predictable performance and low operational overhead.  
The design emphasizes horizontal scalability, isolation between workloads, adaptive throttling, and efficient resource utilization.

---

### Workload Characteristics

Notification workloads exhibit the following patterns:

- **High-volume spikes**  
  Large bursts (e.g., end-of-cycle billing events, bulk campaigns) may result in millions of notifications within minutes.

- **Uneven channel distribution**  
  Some use cases may be SMS-heavy (alerts) while others are email-heavy (reports).

- **Latency tiering**  
  - Real-time: SMS, push  
  - Near real-time: email  
  - Batch: paper mail  

- **Payload variability**  
  Notifications range from lightweight SMS bodies to complex email templates or print-ready PDFs.

- **Multi-tenant usage**  
  Some tenants may generate disproportionately higher traffic, requiring isolation.

These characteristics drive the need for queue-based architecture, per-channel throttling, and elastic scaling.

---

### Scaling Strategies

#### **1. Horizontal Scaling of API Layer**
- Stateless API nodes allow scale-out based on request throughput.
- API Gateways handle rate-limiting, auth, and routing traffic to available nodes.
- Ensures ingestion remains stable even during traffic spikes.

#### **2. Horizontal Scaling of Workers**
Each channel service uses independent worker pools.

- Increase worker count per channel based on:
  - Queue depth  
  - Provider response latency  
  - Time-critical nature (SMS vs email)  

- Allows tuning throughput per channel:
  - Example: 5× more email workers than SMS workers.

#### **3. Per-Channel Queue Isolation**
- Surges in one channel (e.g., SMS) do **not** impact others.
- Prevents a slow provider from creating global backpressure.

#### **4. Priority-Based Processing**
Separate queues (or partitions) for:
- High-priority (alerts)
- Normal notifications
- Bulk/batch workloads

Ensures real-time alerts are not delayed by bulk notifications.

#### **5. Adaptive Throttling**
The system dynamically adjusts throughput based on:
- Provider rate limits  
- Error codes (e.g., 429 Rate Limit Exceeded)  
- Observed latency spikes  
- Provider health checks  

Approach:
- Token bucket / leaky bucket per provider  
- Backoff + retry logic integrated into worker loops

#### **6. Caching Hot Data Paths**
To reduce latency and DB load:
- Preference reads cached (<5ms)
- Routing configs cached
- Template fragments cached
- Provider health cached

Caches are invalidated via TTL and event-driven updates.

---

### Performance Considerations

#### **1. End-to-End Latency Targets**
Each channel type has different acceptable SLAs:

- **SMS:** 100–500ms end-to-end  
- **Push:** <200ms  
- **Email:** 1–5 seconds acceptable  
- **Paper:** minutes to hours (batch workflows)

Routing + rendering optimized to <15–25ms per request.

#### **2. Efficient Template Rendering**
Performance optimizations:
- Compile templates once and cache serialized versions.
- Pre-fetch localized templates.
- Pre-render static fragments; merge dynamic data at runtime.

#### **3. Batching for Slow Channels (Paper Mail)**
- Batch >1000 print jobs into a single PDF generation run.
- Schedule jobs during off-peak hours to reduce compute contention.

#### **4. Provider Selection Based on Performance**
Use provider metrics:
- Latency  
- Error rate  
- Throughput capacity  
- Cost  

Workers select providers dynamically (active-active strategy).

#### **5. Minimizing DB Contention**
- Write-heavy operations (events, logs) sent to append-only stores.
- Read-heavy operations (preferences) routed to distributed KV stores.
- Avoid synchronous DB writes during request ingestion.

---

### Handling Traffic Spikes

To ensure resilience during sudden load surges:

- **Queue backpressure** absorbs surges without dropping requests.
- **Autoscaling** triggered by:
  - Queue depth  
  - Processing latency  
  - Worker CPU usage  
- **Circuit breakers** protect against failing or lagging providers.
- **Load shedding** for low-priority workloads under extreme load.

---

### Capacity Planning & Benchmarks

A scalable notification system should support:

- **Millions of requests per day**  
- **Peak bursts 5–10× baseline traffic**  
- **Sub-millisecond queue enqueue time**  
- **Template rendering throughput of >10k RPS** (with caching)
- **Worker pools scaling to hundreds or thousands of threads/containers**

Capacity planning inputs:
- Notification volume per tenant/time window  
- Provider throughput limits  
- Channel mix (SMS vs email)  
- Template complexity  
- Retry/fallback frequency  

---

### Scalability Diagram

```mermaid
flowchart TD

    subgraph Producers[Producers]
        Billing
        Alerts
        Insights
        Analytics
    end

    Producers --> API[Notification API Gateway]

    API --> LB[Load Balancer / Autoscaling Frontend]
    LB --> ORCH[Orchestration Service]

    subgraph Routing[Routing Components]
        Prefs[Preference Cache]
        Rules[Routing Rules Store]
        Templates[Template Cache]
    end

    ORCH --> Prefs
    ORCH --> Rules
    ORCH --> Templates

    ORCH --> MQ[(Channel Queues - Distributed Message Broker)]

    subgraph SMSGroup[SMS Worker Pool]
        SMS1[SMS Worker 1]
        SMS2[SMS Worker 2]
        SMSN[SMS Worker N (Autoscaled)]
    end

    subgraph EmailGroup[Email Worker Pool]
        EM1[Email Worker 1]
        EM2[Email Worker 2]
        EMN[Email Worker N (Autoscaled)]
    end

    MQ --> SMSGroup
    MQ --> EmailGroup

    SMSGroup --> SMSProv[(SMS Providers - Twilio/Nexmo)]
    EmailGroup --> EmailProv[(Email Providers - SendGrid/AWS SES)]

    subgraph Scaling[Scaling Intelligence]
        QDepth[Queue Depth Monitor]
        WLag[Worker Lag Monitor]
        RPS[Throughput Analyzer]
    end

    QDepth --> HPA[Horizontal Pod Autoscaler]
    WLag --> HPA
    RPS --> HPA

    HPA --> SMSGroup
    HPA --> EmailGroup

    subgraph Stores[Data Stores]
        KV[(KV Store - Preferences)]
        RDB[(Relational DB)]
        Cache[(Distributed Cache)]
        Logs[(Event/Log Store)]
    end

    ORCH --> KV
    ORCH --> RDB
    ORCH --> Cache
    SMSGroup --> Logs
    EmailGroup --> Logs

    subgraph Obs[Observability]
        Metrics
        Traces
        AlertsSys[Alerting System]
    end

    ORCH --> Metrics
    SMSGroup --> Metrics
    EmailGroup --> Metrics

    Metrics --> AlertsSys
    Traces --> AlertsSys
```

The above diagram illustrates:
- API layer scaling independently  
- Orchestration service horizontally scalable  
- Per-channel queues  
- Auto-scaled worker pools  
- Provider throttling modules  
- Observability and real-time metrics influencing autoscaling


---
[→ Next page: Reliability, Fault Tolerance & Delivery Semantics](8_reliability_fault_tolerance_delivery_semantics.md)

[← Table of Contents](../../README.md)