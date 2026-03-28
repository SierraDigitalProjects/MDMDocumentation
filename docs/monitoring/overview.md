# Autonomous Monitoring Architecture

## Architecture Under Observation

The monitoring strategy covers every layer of the BTP Kyma MDM stack:

```
┌──────────────────────── SAP BTP ──────────────────────────────────────┐
│                                                                        │
│  ┌─────────────────── Kyma Runtime ────────────────────────────────┐  │
│  │                                                                  │  │
│  │  Kyma API Gateway (Ory + Istio) ◄── [M1] Istio/Envoy metrics   │  │
│  │         │                                                        │  │
│  │  ┌──────▼────────────────────────────────────────────────────┐  │  │
│  │  │                    MDM Namespace                           │  │  │
│  │  │  mdm-core-api  ◄── [M2] Micrometer /actuator/prometheus   │  │  │
│  │  │  schema-svc    ◄── [M2] Micrometer                        │  │  │
│  │  │  consumer-reg  ◄── [M2] Micrometer                        │  │  │
│  │  │  scheduler-svc ◄── [M2] Quartz metrics + Micrometer       │  │  │
│  │  │  change-tracker◄── [M2] Micrometer                        │  │  │
│  │  └──────┬──────────────────────┬──────────────────┬──────────┘  │  │
│  │         │ write                │ read             │ publish      │  │
│  │  ┌──────▼──┐            ┌──────▼──┐        ┌─────▼──────────┐  │  │
│  │  │ MongoDB │◄──[M3]     │  Redis  │◄──[M4] │  AEM Client    │  │  │
│  │  │(Operator│mongodb_exp │ (BTP    │redis_exp│  (AMQP/REST)   │  │  │
│  │  │on Kyma) │            │  HSO)   │        └────────────────┘  │  │
│  │  └─────────┘            └─────────┘                            │  │
│  │                                                                  │  │
│  │  ┌────────────────────────────────────────────────────────────┐ │  │
│  │  │  Kyma Telemetry (OTel Collector + FluentBit — managed)     │ │  │
│  │  │  MetricPipeline → SAP Cloud Logging / Grafana Cloud        │ │  │
│  │  │  LogPipeline    → SAP Cloud Logging (OpenSearch)           │ │  │
│  │  │  TracePipeline  → Grafana Tempo                            │ │  │
│  │  └────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  SAP AEM ◄──────────── [M5] AEM SEMP API + Solace metrics             │
│  SAP CPI ◄──────────── [M6] CPI Monitoring API + Error Inbox          │
└────────────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
           S/4HANA           CRM           WMS/ERP
         [M7] IDoc/          [M7] REST     [M7] IDoc
         delivery status     response      response
```

**Monitoring probe points:**

| Ref | Layer | What is Monitored |
|---|---|---|
| M1 | Kyma API Gateway | Request rate, latency, error rate per API endpoint |
| M2 | MDM Microservices | JVM, HTTP, business metrics, circuit breaker state |
| M3 | MongoDB on Kyma | Ops/s, replication lag, Change Stream health, connection pool |
| M4 | Redis (BTP HSO) | Cache hit/miss rate, memory, evictions, latency |
| M5 | SAP AEM | Queue depth per consumer queue, message throughput, DLQ count |
| M6 | SAP CPI | iFlow success/failure rate, processing time, Error Inbox count |
| M7 | Satellite systems | CPI delivery outcome (success / retry / DLQ) per iFlow |

---

## Three Pillars + Autonomy Layer

```
┌────────────────────────────────────────────────────────────┐
│                 AUTONOMOUS ACTION LAYER                     │
│  KEDA auto-scale │ K8s self-heal │ Robusta auto-remediate  │
└───────────────────────────┬────────────────────────────────┘
                            │ triggers
┌───────────────────────────▼────────────────────────────────┐
│            INTELLIGENCE / ALERTING LAYER                    │
│  Prometheus Alertmanager │ vmanomaly │ SAP Alert Notif Svc  │
└───────────────────────────┬────────────────────────────────┘
                            │ feeds
┌───────────────────────────▼────────────────────────────────┐
│                  OBSERVABILITY LAYER                        │
│  Kyma Telemetry │ SAP Cloud Logging │ Grafana Tempo         │
│  Prometheus │ AEM SEMP │ CPI Monitoring API                  │
└────────────────────────────────────────────────────────────┘
```

---

## Full Stack

| Capability | Tool | Hosted |
|---|---|---|
| Metrics collection | Prometheus (via Kyma Telemetry MetricPipeline) | Kyma managed |
| Long-term metrics | Thanos or Grafana Cloud | External |
| Logs | SAP Cloud Logging (OpenSearch) via Kyma LogPipeline | BTP managed |
| Traces | Grafana Tempo via Kyma TracePipeline + OTel Java agent | External |
| Dashboards | Grafana | External / Grafana Cloud |
| AEM monitoring | AEM SEMP API + Solace metrics | BTP managed |
| CPI monitoring | CPI Monitoring API + CPI Error Inbox | BTP managed |
| Anomaly detection | VictoriaMetrics vmanomaly | Kyma self-managed |
| Alerting | Prometheus Alertmanager + SAP Alert Notification Service | Kyma + BTP |
| Auto-scaling | KEDA (built-in Kyma) driven by AEM queue depth | Kyma built-in |
| Self-healing | Kubernetes liveness/readiness probes | K8s native |
| Auto-remediation | Robusta runbooks | Kyma self-managed |
| Circuit breaker | Resilience4j (in-app, Spring Boot) | In-process |

---

## Layer 1 — Observability

### Metrics — Kyma MetricPipeline

Kyma's managed OTel Collector scrapes all pods automatically. No manual Prometheus DaemonSet install.

```yaml
apiVersion: telemetry.kyma-project.io/v1alpha1
kind: MetricPipeline
metadata:
  name: mdm-metrics
spec:
  input:
    prometheus:
      enabled: true          # scrapes /actuator/prometheus from all pods
    runtime:
      enabled: true          # K8s runtime metrics (kube-state-metrics)
    istio:
      enabled: true          # Istio/Envoy sidecar metrics (API Gateway layer)
  output:
    otlp:
      endpoint:
        value: https://your-grafana-cloud-otlp-endpoint
      authentication:
        basic:
          user:
            valueFrom:
              secretKeyRef: { name: grafana-secret, key: username }
          password:
            valueFrom:
              secretKeyRef: { name: grafana-secret, key: password }
```

#### Metric Sources per Component

| Component | Exporter / Source | Key Metrics |
|---|---|---|
| `mdm-core-api` | Micrometer `/actuator/prometheus` | HTTP rate, latency p99, error rate, circuit breaker state |
| `schema-service` | Micrometer | Schema validation failure rate |
| `consumer-registry` | Micrometer | Consumer registration count, lookup latency |
| `scheduler-svc` | Micrometer + Quartz JMX | Job last run timestamp, job execution duration, missed jobs |
| `change-tracker` | Micrometer | Field diff computation time, change record write rate |
| MongoDB (on Kyma) | `mongodb_exporter` sidecar | Ops/s, connections, replication lag, Change Stream active |
| Redis (BTP HSO) | `redis_exporter` | Hit rate, miss rate, memory usage, eviction count |
| Kyma API Gateway | Istio Envoy sidecar | Ingress requests/s, latency p95/p99, 4xx/5xx rate |
| Kyma Pods | `kube-state-metrics` | Pod restarts, OOMKills, pending pods, resource saturation |

#### AEM Metrics — SEMP API Polling

AEM (Solace) exposes queue statistics via the SEMP REST API. A lightweight scraper sidecar polls it and exposes Prometheus metrics.

```python
# aem-metrics-scraper — runs as sidecar in monitoring namespace
def scrape_aem_queues():
    queues = ["mdm.crm.customer.realtime.queue",
              "mdm.erp.vendor.scheduled.queue",
              "mdm.wms.material.lastvalue.queue",
              "mdm.audit.all.queue"]
    for queue in queues:
        resp = requests.get(
            f"{AEM_SEMP_URL}/msgVpns/{MSG_VPN}/queues/{queue}",
            auth=(AEM_USER, AEM_PASS)
        )
        data = resp.json()["data"]
        gauge("aem_queue_message_count",  data["msgs"]["count"],   queue=queue)
        gauge("aem_queue_consumer_count", data["consumers"]["count"], queue=queue)
        gauge("aem_queue_dlq_count",      data["msgSpoolUsage"],   queue=queue)
```

#### CPI Metrics — Monitoring API

CPI provides a REST Monitoring API for iFlow execution statistics. Polled every 5 minutes.

```python
# cpi-metrics-scraper
def scrape_cpi_iflows():
    iflows = ["S4_BusinessPartner_To_MDM_Customer",
              "MDM_Customer_To_CRM_Realtime",
              "MDM_Vendor_To_ERP_NightlyBatch",
              "MDM_Material_To_WMS_Realtime"]
    for iflow in iflows:
        resp = requests.get(
            f"{CPI_MONITORING_URL}/MessageProcessingLogs?$filter=IntegrationFlowName eq '{iflow}'",
            headers={"Authorization": f"Bearer {get_token()}"}
        )
        msgs = resp.json()["d"]["results"]
        success = sum(1 for m in msgs if m["Status"] == "COMPLETED")
        failed  = sum(1 for m in msgs if m["Status"] == "FAILED")
        gauge("cpi_iflow_success_total", success, iflow=iflow)
        gauge("cpi_iflow_failed_total",  failed,  iflow=iflow)
```

---

### Logs — SAP Cloud Logging via Kyma LogPipeline

```yaml
apiVersion: telemetry.kyma-project.io/v1alpha1
kind: LogPipeline
metadata:
  name: mdm-logs
spec:
  input:
    application:
      namespaces:
        include: ["mdm"]
      containers:
        exclude: ["istio-proxy"]   # exclude noisy sidecar logs
  filter:
    - tag: mdm.*
      record:
        cluster: kyma-mdm-prod
        environment: production
  output:
    http:
      host:
        valueFrom:
          secretKeyRef: { name: sap-cloud-logging-secret, key: ingest-host }
      user:
        valueFrom:
          secretKeyRef: { name: sap-cloud-logging-secret, key: ingest-user }
      password:
        valueFrom:
          secretKeyRef: { name: sap-cloud-logging-secret, key: ingest-password }
      uri: /ela/ingest/logs/v1/json
      port: "443"
```

All Spring Boot services emit structured JSON logs with `traceId` and `spanId` injected by the OTel agent — enabling log-to-trace correlation in Grafana.

```json
{
  "timestamp": "2024-06-20T14:30:01.234Z",
  "level": "INFO",
  "service": "mdm-core-api",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7",
  "message": "Published eventType=changed objectId=MDM-CUST-001 topic=mdm/customer/changed/v1/S4HANA"
}
```

---

### Traces — Grafana Tempo via Kyma TracePipeline

```yaml
apiVersion: telemetry.kyma-project.io/v1alpha1
kind: TracePipeline
metadata:
  name: mdm-traces
spec:
  output:
    otlp:
      endpoint:
        value: https://your-grafana-tempo-endpoint
      authentication:
        basic:
          user:
            valueFrom:
              secretKeyRef: { name: grafana-secret, key: tempo-user }
          password:
            valueFrom:
              secretKeyRef: { name: grafana-secret, key: tempo-password }
```

End-to-end trace path for an inbound S/4HANA change:

```
CPI iFlow (S4→MDM)
    └── HTTP POST /api/v1/objects/customer          [span: http.server]
            └── MdmCoreService.updateObject()       [span: mdm.update]
                    ├── FieldChangeTracker.diff()   [span: mdm.diff]
                    ├── MongoDB save()              [span: db.mongodb]
                    ├── ChangeRecord save()         [span: db.mongodb]
                    └── AemEventPublisher.publish() [span: messaging.publish]
```

Sampling: 100% for errors and slow requests (>2s), 10% for normal traffic.

---

## Layer 2 — Intelligence and Alerting

### Prometheus Alert Rules — BTP Kyma Specific

```yaml
groups:
- name: mdm-kyma-critical
  rules:

  # MDM API Gateway (Istio)
  - alert: MDMApiGatewayErrorRateHigh
    expr: >
      rate(istio_requests_total{destination_service_name="mdm-core-api",
           response_code=~"5.."}[5m]) /
      rate(istio_requests_total{destination_service_name="mdm-core-api"}[5m]) > 0.05
    for: 3m
    labels: { severity: critical, layer: gateway }
    annotations:
      summary: "MDM API Gateway 5xx rate > 5% for 3 minutes"

  # MDM Core API
  - alert: MDMCoreApiDown
    expr: up{job="mdm-core-api"} == 0
    for: 1m
    labels: { severity: critical, layer: api }
    annotations:
      summary: "mdm-core-api is not reachable — MDM writes and reads are down"

  - alert: MDMCoreApiHighLatency
    expr: histogram_quantile(0.99, rate(http_server_requests_seconds_bucket{
          job="mdm-core-api"}[5m])) > 2
    for: 5m
    labels: { severity: warning, layer: api }
    annotations:
      summary: "MDM API p99 latency > 2s"

  # MongoDB Change Stream (CDC)
  - alert: MongoChangeStreamInactive
    expr: mdm_mongo_changestream_active == 0
    for: 2m
    labels: { severity: critical, layer: cdc }
    annotations:
      summary: "MongoDB Change Stream listener is not active — CDC pipeline broken"

  - alert: MongoReplicationLagHigh
    expr: mongodb_replication_lag_seconds > 30
    for: 5m
    labels: { severity: warning, layer: database }
    annotations:
      summary: "MongoDB replication lag > 30s — risk of stale reads"

  # Redis (BTP HSO)
  - alert: RedisCacheHitRateLow
    expr: >
      rate(redis_keyspace_hits_total[5m]) /
      (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m])) < 0.7
    for: 10m
    labels: { severity: warning, layer: cache }
    annotations:
      summary: "Redis cache hit rate < 70% — MongoDB under unexpectedly high read load"

  - alert: RedisMemoryHigh
    expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.85
    for: 5m
    labels: { severity: warning, layer: cache }
    annotations:
      summary: "Redis memory utilisation > 85% — evictions likely"

  # SAP AEM
  - alert: AemConsumerQueueDepthHigh
    expr: aem_queue_message_count{queue=~"mdm\\..*\\.queue"} > 50000
    for: 15m
    labels: { severity: warning, layer: aem }
    annotations:
      summary: "AEM queue {{ $labels.queue }} has {{ $value }} undelivered messages"

  - alert: AemDlqMessagePresent
    expr: aem_queue_dlq_count{queue=~"mdm\\..*"} > 0
    for: 1m
    labels: { severity: critical, layer: aem }
    annotations:
      summary: "Messages in AEM DLQ for {{ $labels.queue }} — delivery failures need investigation"

  - alert: AemNoConsumerOnQueue
    expr: aem_queue_consumer_count{queue=~"mdm\\..*\\.queue"} == 0
    for: 5m
    labels: { severity: critical, layer: aem }
    annotations:
      summary: "No active consumer on AEM queue {{ $labels.queue }}"

  # SAP CPI
  - alert: CpiIFlowFailureRateHigh
    expr: >
      rate(cpi_iflow_failed_total[10m]) /
      (rate(cpi_iflow_success_total[10m]) + rate(cpi_iflow_failed_total[10m])) > 0.1
    for: 5m
    labels: { severity: critical, layer: cpi }
    annotations:
      summary: "CPI iFlow {{ $labels.iflow }} failure rate > 10%"

  - alert: CpiErrorInboxNonEmpty
    expr: cpi_error_inbox_count > 0
    for: 5m
    labels: { severity: warning, layer: cpi }
    annotations:
      summary: "{{ $value }} messages in CPI Error Inbox — manual review required"

  # Scheduler
  - alert: ScheduledDeltaJobMissed
    expr: mdm_scheduler_last_run_seconds{job_name=~".*delta.*"} > 90000
    for: 0m
    labels: { severity: warning, layer: scheduler }
    annotations:
      summary: "Delta sync job {{ $labels.job_name }} has not run in 25+ hours"

  # Kyma Pod Health
  - alert: MDMPodCrashLooping
    expr: rate(kube_pod_container_status_restarts_total{namespace="mdm"}[15m]) > 0
    for: 5m
    labels: { severity: critical, layer: platform }
    annotations:
      summary: "Pod {{ $labels.pod }} in MDM namespace is crash-looping"
```

### Alert Routing — Alertmanager + SAP Alert Notification Service

Two parallel alert channels:

```yaml
# Alertmanager — routes to Slack / PagerDuty
route:
  receiver: slack-warnings
  group_by: [alertname, layer]
  routes:
  - match: { severity: critical }
    receiver: pagerduty-oncall
    repeat_interval: 30m
  - match: { severity: warning, layer: cpi }
    receiver: slack-integration-team    # CPI issues → integration team, not oncall
  - match: { severity: warning }
    receiver: slack-mdm-alerts

receivers:
- name: pagerduty-oncall
  pagerduty_configs:
  - service_key: ${PAGERDUTY_KEY}
    description: '{{ .CommonAnnotations.summary }}'
- name: slack-mdm-alerts
  slack_configs:
  - channel: '#mdm-alerts'
    title: '[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
    text: '{{ .CommonAnnotations.summary }}'
- name: slack-integration-team
  slack_configs:
  - channel: '#mdm-integration'
    text: 'CPI alert: {{ .CommonAnnotations.summary }}'
```

SAP Alert Notification Service (BTP) is used in parallel for BTP-native alerts (AEM health events, CPI system alerts) that originate within the BTP platform itself and cannot be scraped by Prometheus.

### Anomaly Detection — vmanomaly

`vmanomaly` runs in the Kyma monitoring namespace and self-calibrates against 2 weeks of historical data.

| Signal | Model | What it detects |
|---|---|---|
| MDM object creation rate | z-score | Unusual bulk loads or dead inbound pipelines |
| AEM queue depth per queue | Prophet (seasonal) | Queues growing faster than expected for time of day |
| CPI iFlow processing time | MAD | Sudden increase in integration processing time |
| Redis cache miss rate | z-score | Cache invalidation storm or Redis memory pressure |
| MongoDB change stream events/s | z-score | CDC pipeline stall or unexpected burst |

---

## Layer 3 — Autonomous Action

### Auto-Scaling — KEDA on Kyma

KEDA is built into Kyma. Scale targets are driven by AEM queue depth (primary) and Prometheus metrics (secondary).

```yaml
# Scale consumer-registry based on AEM queue depth
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: mdm-core-api-scaler
  namespace: mdm
spec:
  scaleTargetRef:
    name: mdm-core-api
  minReplicaCount: 2
  maxReplicaCount: 10
  cooldownPeriod: 60
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring.svc.cluster.local:9090
      metricName: http_server_requests_active
      query: sum(rate(http_server_requests_seconds_count{job="mdm-core-api"}[1m]))
      threshold: "50"       # scale up when >50 req/s per replica
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: consumer-router-scaler
  namespace: mdm
spec:
  scaleTargetRef:
    name: consumer-router-service
  minReplicaCount: 2
  maxReplicaCount: 20
  cooldownPeriod: 120
  triggers:
  - type: solace-event-queue
    metadata:
      solaceSempBaseURL: ${AEM_SEMP_URL}
      messageVpn: ${AEM_MSG_VPN}
      queueName: mdm.consumer-router.queue
      messageCountTarget: "500"
      authenticationRef: solace-auth-secret
```

### Self-Healing — Kubernetes Probes

```yaml
# Applied to all MDM Spring Boot services
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 8080 }
  initialDelaySeconds: 30
  failureThreshold: 3
  periodSeconds: 10
  # Kubernetes restarts pod automatically if liveness fails 3 times

readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 8080 }
  initialDelaySeconds: 20
  failureThreshold: 2
  periodSeconds: 5
  # Pod removed from Istio load balancer if readiness fails — no traffic sent to unhealthy pod
```

Spring Boot Actuator health indicators report:
- `db` — MongoDB connectivity
- `redis` — Redis connectivity
- `diskSpace` — JVM heap saturation
- Custom: `aemConnection` — Solace JCSMPSession health

### Auto-Remediation — Robusta Runbooks

Robusta listens for Prometheus alerts and executes automated fix runbooks, posting enriched Slack notifications with metric graphs and action taken.

```python
@action
def restart_mongo_changestream_listener(alert: PrometheusKubernetesAlert):
    """
    Triggered by: MongoChangeStreamInactive
    Fix: Rolling restart of mdm-core-api pod to re-establish Change Stream
    """
    pod = alert.subject
    pod.delete()  # Kubernetes recreates it from Deployment spec
    alert.add_enrichment([
        MarkdownBlock("Auto-restarted mdm-core-api pod to recover MongoDB Change Stream"),
        GraphBlock("mdm_mongo_changestream_active", "Change Stream status")
    ])

@action
def flush_mdm_cache_on_inconsistency(alert: PrometheusKubernetesAlert):
    """
    Triggered by: RedisCacheHitRateLow (sustained)
    Fix: Flush Redis cache for affected object type, force cache rebuild from MongoDB
    """
    object_type = alert.labels.get("object_type", "*")
    redis_client.delete_pattern(f"mdm:{object_type}:*")
    alert.add_enrichment([
        MarkdownBlock(f"Redis cache flushed for objectType={object_type} — will rebuild from MongoDB"),
    ])

@action
def trigger_cpi_error_inbox_notification(alert: PrometheusKubernetesAlert):
    """
    Triggered by: CpiErrorInboxNonEmpty
    Fix: Cannot auto-fix CPI errors (require human decision) — notify integration team with direct link
    """
    alert.add_enrichment([
        MarkdownBlock("CPI Error Inbox has failed messages — manual review required"),
        LinkBlock("Open CPI Error Inbox",
                  f"{CPI_BASE_URL}/itspaces/shell/design/ErrorInbox")
    ])

@action
def scale_up_consumer_router_manually(alert: PrometheusKubernetesAlert):
    """
    Triggered by: AemConsumerQueueDepthHigh (when KEDA has not yet acted)
    Fix: Manually bump replica count as an emergency measure
    """
    patch_deployment("mdm", "consumer-router-service", replicas=10)
    alert.add_enrichment([
        MarkdownBlock("Emergency scale-up: consumer-router-service set to 10 replicas"),
        GraphBlock("aem_queue_message_count", "AEM queue depth")
    ])
```

### Circuit Breaker — Resilience4j (In-App)

Applied at the service layer to prevent cascade failures when MongoDB or Redis are temporarily unavailable.

```java
// MongoDB read — falls back to Redis cache if MongoDB is unhealthy
@CircuitBreaker(name = "mongoDb", fallbackMethod = "fallbackToCache")
@TimeLimiter(name = "mongoDb")
public CompletableFuture<MasterDataObject> getObject(String id) {
    return CompletableFuture.supplyAsync(() -> mongoRepository.findById(id).orElseThrow());
}

public CompletableFuture<MasterDataObject> fallbackToCache(String id, Exception e) {
    log.warn("MongoDB circuit open for id={} — serving from Redis cache", id);
    return CompletableFuture.supplyAsync(() -> redisCache.get(id));
}

// AEM publish — falls back to local retry queue if AEM is temporarily unavailable
@CircuitBreaker(name = "aem", fallbackMethod = "fallbackToLocalRetryQueue")
public void publishToAem(MdmEvent event) {
    aemEventPublisher.publish(event);
}

public void fallbackToLocalRetryQueue(MdmEvent event, Exception e) {
    log.warn("AEM circuit open — queuing event locally for retry: {}", event.getEventId());
    localRetryQueue.add(event);
}
```

Circuit breaker states exported to Prometheus via Micrometer:

```
resilience4j_circuitbreaker_state{name="mongoDb", state="closed"}  1
resilience4j_circuitbreaker_state{name="aem",     state="open"}    1
```

---

## Key Dashboards

### Dashboard 1 — End-to-End MDM Pipeline Health

| Panel | Source | Alert threshold |
|---|---|---|
| Inbound iFlow success rate (CPI) | CPI Monitoring API | < 95% triggers warning |
| AEM queue depth per consumer queue | AEM SEMP API | > 50K triggers warning |
| AEM DLQ count | AEM SEMP API | > 0 triggers critical |
| MDM API p99 latency | Istio metrics | > 2s triggers warning |
| MDM API error rate | Istio metrics | > 5% triggers critical |
| MongoDB Change Stream active | Custom metric | = 0 triggers critical |
| Redis cache hit rate | redis_exporter | < 70% triggers warning |
| Scheduler last run per job | Quartz metrics | > 25h triggers warning |

### Dashboard 2 — AEM + CPI Integration Health

- Per-queue message depth over time (all MDM queues)
- Per-iFlow execution success/failure rate (all CPI iFlows)
- CPI Error Inbox count trend
- AEM consumer connection count per queue
- End-to-end latency: S/4HANA push → MDM golden record → AEM published

### Dashboard 3 — Consumer Fan-out Health

- Per-consumer delivery success rate
- Field-change trigger rate (how often each consumer is notified)
- Consumers with zero deliveries in last 24 hours (possible dead consumer)
- Scheduled batch consumer last execution time

### Dashboard 4 — Data Quality

- Schema validation failure rate per object type
- Duplicate detection rate (potential merge candidates)
- Field change frequency heatmap (which fields change most often)
- Change record volume by module per day

---

## Deployment Layout — Kyma

```
SAP BTP Kyma Cluster
│
├── mdm namespace
│   ├── mdm-core-api
│   ├── schema-service
│   ├── consumer-registry
│   ├── scheduler-service
│   ├── change-tracker
│   ├── mongodb-exporter (sidecar/standalone)
│   ├── redis-exporter   (sidecar/standalone)
│   ├── aem-metrics-scraper
│   └── cpi-metrics-scraper
│
├── monitoring namespace
│   ├── Prometheus + Alertmanager
│   ├── vmanomaly (anomaly detection)
│   ├── Robusta   (auto-remediation runbooks)
│   └── Grafana   (dashboards)
│
└── kyma-system namespace (managed by SAP)
    ├── Kyma Telemetry (OTel Collector + FluentBit)  ──► SAP Cloud Logging
    ├── Kyma TracePipeline                            ──► Grafana Tempo
    ├── Kyma MetricPipeline                           ──► Grafana Cloud
    └── KEDA operator
```
