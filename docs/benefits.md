# Benefits

## Business Benefits

| Benefit | Detail | Who Gains |
|---|---|---|
| **Elimination of duplicate records** | Match-and-merge engine prevents duplicate Business Partners, Materials, and Vendors from propagating across systems | All teams consuming master data |
| **Faster satellite system onboarding** | New consumer systems register field interest and receive events via CPI — no custom point-to-point integration build | Integration team, new site teams |
| **Reduced manual data reconciliation** | Satellite systems no longer need to reconcile master data manually against S/4HANA exports | Site Operations, Finance, SCM |
| **Consistent pricing and customer data** | BP Master and Pricing Condition Records are sourced from a single golden record — no pricing discrepancies from stale data | Sales, Finance |
| **Audit-ready compliance** | Immutable change log with who changed what, when, and through which approval chain — ready for internal and external audit | Compliance, Legal |
| **Self-service data extension** | Satellite system teams can add their own extension attributes to shared objects without raising change requests to the Data Team | CRM team, ERP team, WMS team |

---

## Technical Benefits

| Benefit | Detail |
|---|---|
| **Decoupled integration** | Satellite systems subscribe to AEM topics — no direct S/4HANA coupling. Adding a new consumer does not change the MDM core |
| **Schema evolution without migration** | MongoDB dynamic schema allows object definitions to evolve without breaking existing consumers |
| **Field-level change tracking** | Consumers are triggered only when fields they care about change — reduces unnecessary processing in satellite systems |
| **Read-through caching** | Redis cache absorbs read traffic spikes — MongoDB is not hit directly by all consumer read requests |
| **Event replay capability** | AEM persistent queues allow new consumers to receive historical events on first connection — bootstrapping without a full dump |
| **Compacted master data topics** | AEM Last-Value Queues ensure consumers always receive the latest object state — no need to reconcile intermediate changes |
| **Site isolation** | RBAC enforced at data level — site-scoped objects cannot be modified by coordinators from other sites |
| **Autonomous self-healing** | Kubernetes probes, KEDA auto-scaling, and Robusta runbooks resolve common failure modes without human intervention |

---

## Cost Benefits

### Integration Cost Reduction

| Integration Type | Before MDM | After MDM | Saving |
|---|---|---|---|
| Adding a new satellite system | Build custom S/4HANA → System adapter (~20 days) | Register consumer in MDM + build one CPI iFlow (~5 days) | ~75% per integration |
| Master data format transformation | Each system maintains its own transformation logic | CPI iFlow mapping handles once, all consumers benefit | Maintenance centralised |
| Error handling and retry | Each integration team builds its own retry logic | CPI built-in retry, Error Inbox, DLQ — reused across all iFlows | Shared infrastructure |
| SAP connectivity | Custom RFC/BAPI adapter per system | BTP Connectivity Service + CPI adapter — shared | Single connection |

### Development Cost Reduction (vs AKS Baseline)

| Stack Choice | Effort Reduction vs AKS + Kafka |
|---|---|
| BTP Kyma + Kafka | ~25–35% |
| BTP Kyma + AEM + CPI | ~40–45% |

See [Event Broker — Effort Reduction](technical-architecture/event-broker.md) for the full breakdown.

### Operational Cost Reduction

| Area | Saving |
|---|---|
| Platform infrastructure | Managed Kyma, managed AEM, managed CPI — no Kubernetes admin overhead for core services |
| Incident resolution time | AI-assisted root cause analysis reduces average MTTR from hours to minutes |
| Manual data fixes | Data quality rules prevent bad data entering MDM — fewer downstream reconciliation incidents |
| On-call burden | Autonomous monitoring auto-resolves known failure patterns — fewer overnight pages |

---

## Risk Reduction

| Risk Mitigated | How MDM Addresses It |
|---|---|
| **Stale master data causing business decisions on wrong data** | Event-driven propagation with < 5 min latency for critical objects |
| **No audit trail for data changes** | Immutable change record per object version, full approval chain stored |
| **Data loss during system outages** | AEM persistent queues + MongoDB replica set — no in-flight message lost |
| **Uncontrolled data extension by shadow IT** | RBAC-controlled extension namespaces — extensions visible to all but writable only by authorised role |
| **Compliance failure on EH&S data** | EH&S objects (MSDS, Hazardous Materials) governed through MDM with e-signature-ready approval workflow |
| **HR data leaking to wrong consumers** | Field-level RBAC — HR extension attributes only readable by systems with HR scope grant |
