# Compute Options — BTP Kyma vs Azure AKS

## Decision Guide

| Question | Points to |
|---|---|
| Are your lead systems SAP (S/4HANA, MDG, ECC)? | **BTP Kyma** |
| Do you want fastest time-to-production on platform setup? | **BTP Kyma** |
| Does your team have existing BTP experience? | **BTP Kyma** |
| Are lead systems non-SAP and heterogeneous? | **AKS** |
| Do you need maximum infrastructure control? | **AKS** |
| Do you want to avoid BTP licensing/credit complexity? | **AKS** |

**Hybrid recommendation:** BTP Kyma for the MDM runtime + SAP Integration Suite (CPI) for SAP connectivity + MongoDB Atlas on Azure (linked to BTP). This is the most production-ready combination for an SAP-connected landscape.

---

## Option A — SAP BTP Kyma (Recommended for SAP Landscapes)

Kyma is a managed Kubernetes runtime (Gardener-based) with pre-installed modules that replace components you would otherwise build and operate.

### What Kyma Provides Out of the Box

| Kyma Module | What it Replaces | Effort Saved |
|---|---|---|
| Kyma API Gateway (Istio + Ory) | Kong setup + configuration | High |
| SAP IAS + XSUAA binding | Keycloak install + realm + Spring Security wiring | Very High |
| Kyma Telemetry (OTel + FluentBit) | Manual OTel Collector + Promtail setup | High |
| Kyma Application Connector | Custom SAP system integration layer | Very High |
| BTP Service Binding (SBOM operator) | Manual secret/config wiring for external services | Medium |
| Istio Service Mesh | Manual mTLS, traffic policies, circuit breaker infra | High |
| SAP Alert Notification Service | Alertmanager routing configuration | Medium |
| SAP Cloud Logging (OpenSearch) | Loki setup + retention configuration | Medium |
| BTP Connectivity Service | VPN/tunnel to on-premise S/4HANA | Very High |
| KEDA (built-in) | Manual KEDA Helm install + config | Medium |

### BTP Kyma Architecture

```
┌──────────────────────── SAP BTP ─────────────────────────────────────┐
│                                                                        │
│  SAP IAS ──► XSUAA ──► Kyma API Gateway (Ory Oathkeeper + Istio)     │
│                                │                                       │
│              ┌─────────────────▼───────────────────────────────────┐  │
│              │                Kyma Runtime (K8s)                    │  │
│              │                                                      │  │
│              │  ┌────────────────────────────────────────────────┐ │  │
│              │  │                  MDM Namespace                  │ │  │
│              │  │  mdm-core-api    schema-service                 │ │  │
│              │  │  consumer-registry   fanout-router              │ │  │
│              │  │  scheduler-service   read-cache-service         │ │  │
│              │  └───────┬──────────────────────┬──────────────────┘ │  │
│              │          │ write                 │ publish            │  │
│              │  ┌───────▼───┐           ┌──────▼───────────────┐   │  │
│              │  │ MongoDB   │           │ AEM Client (AMQP)    │   │  │
│              │  │ (Operator)│           └──────┬───────────────┘   │  │
│              │  └───────────┘                  │                   │  │
│              │  ┌───────────┐                  │                   │  │
│              │  │  Redis    │                  │                   │  │
│              │  │ (BTP HSO) │                  │                   │  │
│              │  └───────────┘                  │                   │  │
│              └─────────────────────────────────┼───────────────────┘  │
│                                                │                       │
│  ┌─────────────────────────────────────────────▼──────────────────┐   │
│  │              SAP Advanced Event Mesh (AEM)                      │   │
│  └─────────────────────────────────────────────┬──────────────────┘   │
│                                                │                       │
│  ┌─────────────────────────────────────────────▼──────────────────┐   │
│  │              SAP Cloud Integration (CPI)                        │   │
│  │  iFlow: MDM → CRM    iFlow: MDM → ERP    iFlow: S4 → MDM       │   │
│  └─────────────────────────────────────────────┬──────────────────┘   │
│                                                │                       │
│  BTP Connectivity Service ◄── Kyma App Connector                       │
└──────────────────────────────────────┬────────────────────────────────┘
                                       │ secure tunnel
                              ┌────────▼────────┐
                              │  On-Premise      │
                              │  S/4HANA / ECC   │
                              └─────────────────┘
```

### Effort Reduction vs AKS Baseline

| Area | AKS Effort | BTP Kyma Effort | Reduction |
|---|---|---|---|
| RBAC / Auth | High | Low | ~65% |
| API Gateway | Medium | Very Low | ~55% |
| SAP system inbound connectivity | Very High | Low | ~70% |
| Service mesh / mTLS | Medium | None (built-in) | ~80% |
| Observability pipeline | High | Low | ~50% |
| Redis provisioning + ops | Medium | Very Low | ~60% |
| Infrastructure / K8s setup | High | Low (managed) | ~60% |
| Alert routing | Medium | Low | ~45% |
| Business logic (MDM core) | — | — | 0% (same) |
| Kafka/AEM setup | Medium | Medium (Strimzi) / Low (AEM) | ~10–40% |

**Overall platform/infra effort reduction: ~50–55%**
**Overall project effort reduction (including business logic): ~25–35%**

With CPI + AEM replacing Strimzi Kafka + custom integration code, the overall project effort reduction increases to **~40–45%** vs AKS baseline.

### BTP Kyma Trade-offs

| Risk | Mitigation |
|---|---|
| MongoDB not a native BTP service | Use MongoDB Atlas via Azure Marketplace linked to BTP, or deploy Community Operator on Kyma |
| BTP service costs add up | IAS, XSUAA, Connectivity, Cloud Logging, AEM are metered — plan BTP credits carefully |
| Kyma version upgrades managed by SAP | Less control over K8s version — plan for compatibility testing |
| Vendor lock-in on BTP services | Abstract IAS/XSUAA behind an interface to maintain portability |
| AEM throughput pricing | Solace charges by connection and throughput tier — model your volume |

---

## Option B — Azure AKS

Full Kubernetes flexibility, no BTP dependencies. All components self-managed.

### AKS Stack Differences

| Component | BTP Kyma | Azure AKS |
|---|---|---|
| IAM / RBAC | SAP IAS + XSUAA | Keycloak (self-managed) |
| API Gateway | Kyma API Gateway (Ory + Istio) | Kong (self-managed) |
| Event broker | SAP AEM (managed) or Strimzi | Strimzi Kafka or Redpanda |
| Observability | Kyma Telemetry + SAP Cloud Logging | Prometheus + Loki + Tempo (self-managed) |
| Redis | BTP Hyperscaler Option Service | Redis Stack on AKS (StatefulSet or Azure Cache) |
| SAP connectivity | BTP Connectivity Service | Custom SAP Cloud Connector setup |
| Service mesh | Istio (built-in) | Istio (manual Helm install) |

### AKS Architecture

```
Azure AKS
├── MDM Namespace
│   ├── mdm-core-api (Spring Boot)
│   ├── schema-service
│   ├── consumer-registry
│   ├── fanout-router
│   └── scheduler-service
├── Data Namespace
│   ├── MongoDB StatefulSet (Community Operator)
│   └── Redis Stack StatefulSet
├── Messaging Namespace
│   ├── Strimzi Kafka OR Redpanda
│   ├── Apicurio Schema Registry
│   └── Debezium Kafka Connect
├── Platform Namespace
│   ├── Keycloak
│   ├── Kong API Gateway
│   └── KEDA
└── Monitoring Namespace
    ├── Prometheus + Thanos
    ├── Loki + Promtail
    ├── Tempo
    └── Grafana
```

---

## Kyma Serverless Functions

Kyma Serverless (Node.js / Python) can replace lightweight microservices, reducing Docker image count and deployment overhead.

Good candidates for Kyma Functions:

| Function | Replaces |
|---|---|
| Consumer field-interest filter | Fanout router logic for simple cases |
| Cache invalidation handler | Separate cache invalidation microservice |
| Schema validation webhook | Validation microservice |
| Webhook delivery retry | Custom retry service |

```javascript
// Kyma Function: field-interest filter
module.exports = {
  main: async function(event, context) {
    const { objectType, changedFields } = event.data;
    const consumers = await getInterestedConsumers(objectType, changedFields);
    await Promise.all(consumers.map(c => notifyConsumer(c, event.data)));
    return { notified: consumers.length };
  }
}
```
