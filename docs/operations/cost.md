# Cost of Operations

This page models the ongoing operational cost of running the Sierra Aura on BTP Kyma. All figures are estimates — actual costs depend on usage volume, BTP contract terms, and selected service plans.

---

## Cost Categories

```
Total Monthly Cost
├── BTP Platform Services         (AEM, CPI, XSUAA, Cloud Logging, Redis HSO)
├── Compute (Kyma Runtime)        (node pool sizing)
├── Database (MongoDB)            (Atlas or self-managed)
├── AI / LLM Operations           (Claude API — autonomous monitoring)
├── Observability                 (Grafana Cloud or self-managed)
└── People                        (Data Team, Site Coordinators, BTP Admin)
```

---

## BTP Platform Services

These are managed BTP services billed against BTP cloud credits.

| Service | Plan | Basis | Est. Monthly Cost |
|---|---|---|---|
| **SAP Advanced Event Mesh (AEM)** | Default | Per connection + message throughput | $800–$2,500 |
| **SAP Cloud Integration (CPI)** | Standard | Per integration flow execution | $500–$1,500 |
| **SAP IAS + XSUAA** | Application | Per active user / month | $200–$600 |
| **BTP Connectivity Service** | Standard | Per on-premise connection | $100–$300 |
| **SAP Cloud Logging** | Development/Standard | Per GB ingested | $100–$400 |
| **Redis (BTP Hyperscaler Option)** | Standard | Per GB RAM | $150–$500 |
| **Total BTP Services** | | | **$1,850–$5,800/month** |

!!! warning "AEM Pricing"
    AEM (Solace PubSub+) pricing is sensitive to the number of persistent connections and message throughput tier. Model your peak object change volume and consumer count carefully before selecting an AEM plan. At high object change rates (> 1M events/day), upgrade to the Enterprise tier.

---

## Compute — Kyma Runtime

Kyma runs on a managed Kubernetes node pool. Sizing depends on the number of MDM microservices and message throughput.

| Environment | Node Type | Nodes | Est. Monthly Cost |
|---|---|---|---|
| **Development** | Standard_D4s_v3 (4 vCPU, 16GB) | 2 | ~$250 |
| **Production (Phase 1–2)** | Standard_D8s_v3 (8 vCPU, 32GB) | 3 | ~$900 |
| **Production (Phase 3–5, full load)** | Standard_D8s_v3 | 5 | ~$1,500 |

Node costs depend on the underlying hyperscaler (Azure / GCP) negotiated by SAP for Kyma — BTP Kyma nodes are billed at cloud provider rates within the BTP contract.

---

## Database — MongoDB

=== "MongoDB Atlas (Recommended)"

    | Tier | Spec | Est. Monthly Cost |
    |---|---|---|
    | Development | M10 (2 vCPU, 2GB RAM) | ~$60 |
    | Production — Phase 1–2 | M30 (2 vCPU, 8GB RAM) | ~$200 |
    | Production — Phase 3–5 | M50 (8 vCPU, 32GB RAM) | ~$700 |
    | Production — High Volume | M80 (32 vCPU, 128GB RAM) | ~$2,000 |

    Atlas on Azure can be linked directly to BTP via the Azure Marketplace — service credentials injected as a Kyma ServiceBinding.

=== "MongoDB Community on Kyma"

    Self-managed via Community Operator on Kyma. No licensing cost, but requires a dedicated StatefulSet with persistent volumes.

    | Spec | Storage (Premium SSD) | Est. Monthly Cost |
    |---|---|---|
    | Dev (3-node replica) | 100GB | ~$80 |
    | Prod (3-node replica) | 500GB | ~$300 |
    | Prod (3-node sharded) | 2TB | ~$900 |

---

## AI / LLM Operations — Claude API

AI-assisted monitoring cost scales with alert volume. See [AI in MDM — Cost Economics](../ai/overview.md) for the full token model.

| Scenario | Events Analysed/day | Est. Monthly Cost |
|---|---|---|
| Minimal — critical alerts only | 5 | ~$2 |
| Standard — warnings + critical | 20 | ~$6 |
| Full autonomous — all log anomalies | 500 | ~$80 |
| Incident month — major degradation events | +100/incident | +$24 per incident |

**Recommendation:** Start with Standard tier (critical + warning alerts only). Expand to full autonomous monitoring once baseline anomaly patterns are established.

---

## Observability

=== "Grafana Cloud (Recommended)"

    | Plan | Included | Est. Monthly Cost |
    |---|---|---|
    | Free | 10K metrics series, 50GB logs | $0 |
    | Pro | 100K metrics series, 500GB logs | ~$200–$500 |
    | Advanced | Unlimited + SLA | Custom pricing |

    Kyma MetricPipeline and LogPipeline ship directly to Grafana Cloud via OTLP — no self-managed Prometheus/Loki required.

=== "Self-Managed (Prometheus + Loki on Kyma)"

    No licensing cost, but adds operational overhead for Prometheus, Thanos, Loki, and Grafana on the Kyma cluster.

    | Component | Est. Monthly Compute Cost |
    |---|---|
    | Prometheus + Thanos | ~$80 |
    | Loki + Promtail | ~$60 |
    | Grafana | ~$30 |
    | **Total** | **~$170** |

---

## People Costs

Platform costs are a fraction of total programme cost. People are the largest ongoing operational expense.

| Role | Responsibility | FTE / Allocation |
|---|---|---|
| **MDM Platform Engineer** | Kyma operations, CPI iFlow development, service upgrades | 0.5 FTE ongoing |
| **Data Steward — Central** | Create/approve centrally managed objects, data quality | 1.0 FTE ongoing |
| **Site Data Coordinator** (per site) | Locally managed object maintenance | 0.25 FTE per site |
| **BTP Administrator** | Service provisioning, XSUAA, binding management | 0.25 FTE ongoing |
| **MDM Support (L1/L2)** | CPI Error Inbox triage, consumer registration, user issues | 0.5 FTE ongoing |

At a loaded cost of $800/day (full-time developer equivalent):

| Sites | FTE Required | Est. Monthly People Cost |
|---|---|---|
| 1–3 sites | ~2.5 FTE | ~$40,000 |
| 4–8 sites | ~3.5 FTE | ~$56,000 |
| 9–15 sites | ~5 FTE | ~$80,000 |

---

## Total Cost of Operations Summary

### Phase 1–2 (Foundation + Fan-out, 1–3 sites)

| Category | Monthly |
|---|---|
| BTP Platform Services | $1,850–$3,000 |
| Kyma Compute (3 nodes) | $900 |
| MongoDB Atlas M30 | $200 |
| Grafana Cloud Pro | $300 |
| Claude API (Standard) | $6 |
| **Infrastructure Total** | **~$3,250–$4,400/month** |
| People (2.5 FTE) | ~$40,000/month |
| **Grand Total** | **~$43,000–$44,500/month** |

### Phase 3–5 (Full Platform, 8 sites)

| Category | Monthly |
|---|---|
| BTP Platform Services | $3,000–$5,800 |
| Kyma Compute (5 nodes) | $1,500 |
| MongoDB Atlas M50 | $700 |
| Grafana Cloud Pro | $400 |
| Claude API (Full Autonomous) | $80 |
| **Infrastructure Total** | **~$5,700–$8,500/month** |
| People (4 FTE) | ~$64,000/month |
| **Grand Total** | **~$70,000–$72,500/month** |

---

## Cost vs Value

| Investment | Estimated Saving |
|---|---|
| MDM platform infrastructure ($5K–$9K/month) | Replaces point-to-point integration maintenance (~$15K–$25K/month across all satellite systems) |
| AI monitoring ($80/month) | Saves ~$2,000+/month in developer on-call time for incident diagnosis and resolution |
| AI-assisted development (~$14 per build cycle) | Replaces ~$6,400–$9,600 of boilerplate development per release cycle |
| CPI + AEM vs custom integration (~40% effort reduction) | For a 6-month build at $800/day with a 5-person team: saves ~$80,000–$120,000 in development cost |

---

## Cost Optimisation Levers

| Lever | Potential Saving | Trade-off |
|---|---|---|
| Use MongoDB Community on Kyma instead of Atlas | $200–$700/month | Self-managed ops; no managed backups |
| Downgrade AEM to lower throughput tier off-peak | $300–$600/month | Queue depth may grow overnight |
| Use `claude-haiku-4-5` for log triage instead of Opus | ~80% LLM cost reduction | Lower accuracy on complex incidents |
| Self-managed Prometheus + Loki instead of Grafana Cloud | $0 vs $300–$500/month | Additional Kyma node overhead |
| Reduce CPI iFlow polling frequency for batch consumers | Reduced CPI execution cost | Longer delivery latency for batch |
