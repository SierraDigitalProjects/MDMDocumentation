# Event Broker — AEM vs Kafka vs Kyma Eventing

## Comparison Matrix

| Capability | Kyma Eventing (NATS) | Strimzi Kafka / Redpanda | SAP AEM (Solace) |
|---|---|---|---|
| Fan-out to multiple consumers | Yes | Yes | Yes |
| At-least-once delivery | Yes | Yes | Yes |
| **Compacted topics (last-value per key)** | **No** | **Yes** | **Yes (Last-Value Queue)** |
| **Scheduled / batched delivery** | **No** | **Yes** | **Yes (via CPI Timer)** |
| Consumer lag visibility | No | Yes (native) | Yes (AEM console) |
| Field-level consumer filtering | Custom code | Custom code | Custom code (same) |
| Message replay / rewind | Limited (time-based) | Yes (offset-based) | Limited |
| KEDA autoscaling on lag | NATS JetStream scaler | Kafka scaler (mature) | Solace KEDA scaler |
| Dead letter queue | Basic | Mature | AEM DLQ (built-in) |
| Schema registry | No | Apicurio / Confluent | No (payload opaque) |
| **High throughput** | ~500K msg/s | Millions/s | Millions/s (Solace) |
| Operational complexity | None (managed) | Medium (Strimzi) | None (managed BTP) |
| Kafka client compatibility | No | Yes | No (AMQP/Solace API) |
| SAP ecosystem fit | Native on Kyma | Neutral | Native on BTP |
| Open standard | CloudEvents | Kafka protocol | Solace/AMQP |

---

## Why Kyma Eventing Is Not Sufficient for MDM

### Gap 1 — No Compacted Topics

Master data is **stateful**. If a consumer is offline for two days and reconnects, it should receive only the **latest state** per object — not 48 hours of intermediate changes.

=== "Kafka / AEM (correct)"

    ```
    Object MDM-CUST-001 changed 50 times over 2 days
        │
        ▼ Kafka log compaction / AEM Last-Value Queue
    Consumer receives: MDM-CUST-001 @ version 50 only ✓
    ```

=== "Kyma Eventing (problem)"

    ```
    Object MDM-CUST-001 changed 50 times over 2 days
        │
        ▼ NATS — no compaction
    Consumer receives: all 50 intermediate states
    Consumer must reconcile them in application code ✗
    ```

### Gap 2 — No Scheduled Delivery

Requirement 5 states: consumers must be able to choose **scheduled delivery** (e.g., nightly batch) instead of real-time.

Kyma Eventing pushes immediately — there is no mechanism to hold events until a consumer's scheduled window. Building a buffer service in front of the consumer effectively rebuilds a queue, which is the same effort as deploying Kafka or AEM.

Kafka solves this naturally — the consumer polls on its own schedule from its last committed offset:

```
Scheduler triggers Consumer A at 02:00
    │
    ▼ Consumer polls Kafka from last committed offset
    │ reads all changes since last night in one batch
    ▼
Processes batch, commits offset
```

### Verdict

> Kyma Eventing is **not a replacement** for Kafka or AEM in this MDM architecture.

---

## Recommended Hybrid

Use Kyma Eventing and the external broker **together** — each for what it does well:

```
MDM Core API
    │
    ├──► Kyma Eventing (NATS)         Internal events only
    │        ├── Cache invalidation
    │        ├── Schema change notify
    │        └── Internal service triggers
    │
    └──► AEM / Kafka                  External consumer fan-out
             ├── Satellite system delivery
             ├── Scheduled batch consumers
             ├── Compacted master data topics
             └── Consumer lag tracking
```

---

## AEM vs Kafka — Decision

=== "Choose AEM"

    - Majority of consumers are SAP systems
    - CPI pre-built adapters available — eliminates custom integration code
    - Team has BTP/SAP integration experience
    - Managed service preferred (no Strimzi operator to run)
    - BTP credits / budget available

=== "Choose Kafka (Redpanda)"

    - Non-SAP or mixed consumer landscape
    - Need Kafka ecosystem (Debezium, KEDA Kafka scaler, Schema Registry, Kafka Connect)
    - Want open-standard protocol
    - Budget-constrained (no AEM licensing)
    - Team stronger in Kafka tooling

=== "Choose Both"

    - AEM for SAP-to-SAP fan-out (use CPI pre-built adapters)
    - Redpanda for non-SAP high-volume consumers
    - MDM Core API publishes to both from the same change event

---

## Effort Reduction: Kyma + AEM + CPI vs Kyma + Kafka

Compared to the AKS + Strimzi Kafka + custom integration baseline:

| Area | Kyma + Kafka | Kyma + AEM + CPI | Additional Reduction |
|---|---|---|---|
| Event broker setup / ops | Medium (Strimzi operator) | None (managed AEM) | +40% |
| SAP inbound integration | Low (BTP Connectivity) | Very Low (CPI pre-built content) | +25% |
| Outbound integration per system | High (custom consumer code) | Low (CPI iFlow per system) | +50% |
| Retry / error handling | Custom code | Built-in CPI | +60% |
| Scheduled delivery | Custom scheduler + Kafka | CPI Timer iFlow | +40% |
| Consumer monitoring | Custom Prometheus scraper | CPI + AEM consoles | +35% |
| Schema / format transformation | Custom code | CPI mapping editor | +55% |

**Overall project effort reduction vs AKS baseline: ~40–45%**
*(vs ~25–35% with Kyma + Kafka alone)*

---

## What Still Requires Custom Code

AEM + CPI does not eliminate MDM business logic. These components are always custom-built regardless of the event broker choice:

| Component | Why It Cannot Be Replaced |
|---|---|
| MDM Core API (Spring Boot) | Business logic, schema engine, RBAC enforcement |
| Schema / Object Definition Service | MDM-specific, no off-the-shelf solution |
| Consumer Registry Service | MDM-specific configuration store |
| Field-level change tracker | Field diff computation is always custom |
| Read-through cache (Redis) | Cache layer logic and invalidation strategy |
| MongoDB data model | Custom per MDM object definitions |

---

## Trade-offs

| Trade-off | Detail |
|---|---|
| **Cost** | AEM and CPI are licensed BTP services — factor credits carefully. AEM charges by connection tier and message throughput; high-volume MDM deployments should model costs before committing |
| **CPI iFlow per consumer** | Each satellite system needs its own iFlow — manageable, but adds CPI development overhead as the consumer count grows |
| **AEM is not Kafka-compatible** | Cannot use Kafka client libraries, Debezium Kafka Connect, or the KEDA Kafka scaler — requires AMQP client (Apache Qpid or Solace Java API) and the KEDA Solace scaler instead |
| **Vendor dependency** | Deeper SAP ecosystem lock-in vs Kafka which is an open standard with a broad tooling ecosystem |
| **AEM throughput pricing** | Solace charges by connection and throughput tier — model your peak object change volume before selecting a plan |

---

## Final Recommendation

!!! success "Use AEM + CPI if:"
    - Majority of satellite systems are SAP (S/4HANA, SuccessFactors, Ariba)
    - Your team has CPI iFlow development experience
    - BTP credits / budget is available and scoped
    - You want minimal operational responsibility for the event broker
    - You want to leverage CPI's built-in retry, error inbox, and transformation tooling

!!! info "Stick with Strimzi / Redpanda if:"
    - Satellite systems are mostly non-SAP or highly heterogeneous
    - Budget is constrained and open-source is preferred
    - Your team is stronger in the Kafka / JVM ecosystem
    - You need Debezium Kafka Connect, Kafka Schema Registry, or KEDA Kafka scaler
    - You want open-standard event streaming with no vendor lock-in

---

## AEM Topic Design for MDM

AEM (Solace) uses `/`-separated topic strings with wildcard support (`*` = one level, `>` = all levels below).

```
Topic pattern: mdm/{objectType}/{eventType}/{version}/{leadSystem}

Examples:
  mdm/customer/created/v1/S4HANA
  mdm/customer/changed/v1/S4HANA
  mdm/customer/deleted/v1/CRM
  mdm/vendor/changed/v1/S4HANA
  mdm/material/changed/v1/EWM
```

### Queue Definitions

| Queue Name | Type | Subscriber Wildcard | Consumer |
|---|---|---|---|
| `mdm.crm.customer.realtime.queue` | Persistent | `mdm/customer/>` | CRM system (real-time CPI iFlow) |
| `mdm.erp.vendor.scheduled.queue` | Persistent | `mdm/vendor/>` | ERP system (nightly CPI iFlow) |
| `mdm.wms.material.lastvalue.queue` | Last-Value | `mdm/material/>` | WMS (state sync, latest only) |
| `mdm.audit.all.queue` | Persistent | `mdm/>` | Audit / compliance service |
| `mdm.cache.invalidation.queue` | Non-persistent | `mdm/>` | Internal Redis invalidation |

---

## Kafka Topic Design for MDM

```
Topic per object type — compacted:
  mdm.customer.events        (cleanup.policy=compact)
  mdm.vendor.events          (cleanup.policy=compact)
  mdm.material.events        (cleanup.policy=compact)

Key:   objectId              (enables per-key compaction)
Value: MdmEvent JSON         (full object snapshot + changedFields[])

Consumer groups:
  mdm-crm-consumer-group     → mdm.customer.events
  mdm-erp-consumer-group     → mdm.vendor.events
  mdm-wms-consumer-group     → mdm.material.events
  mdm-audit-consumer-group   → mdm.*  (all topics)
```

### Redpanda vs Kafka

**Redpanda is the strongest single recommendation** if you choose the Kafka path:

| | Kafka (Strimzi) | Redpanda |
|---|---|---|
| Protocol | Kafka protocol | Kafka protocol (compatible) |
| Runtime | JVM | C++ (no JVM) |
| ZooKeeper/KRaft | KRaft required | Not needed |
| Throughput per node | High | ~10x higher |
| Operational complexity | Medium | Low (single binary) |
| Kafka client compatibility | Native | Full (drop-in replacement) |
| Debezium / Kafka Connect | Yes | Yes |
| KEDA scaler | Yes | Yes |
