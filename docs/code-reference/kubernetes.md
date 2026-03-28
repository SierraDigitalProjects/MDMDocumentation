# Kubernetes & Kyma Manifests

## BTP Service Bindings

### AEM Service Instance and Binding

```yaml
# aem-service-instance.yaml
apiVersion: services.cloud.sap.com/v1
kind: ServiceInstance
metadata:
  name: mdm-aem-instance
  namespace: mdm
spec:
  serviceOfferingName: enterprise-messaging
  servicePlanName: default
  parameters:
    options:
      management: true
      messagingrest: true
---
apiVersion: services.cloud.sap.com/v1
kind: ServiceBinding
metadata:
  name: mdm-aem-binding
  namespace: mdm
spec:
  serviceInstanceName: mdm-aem-instance
  secretName: mdm-aem-secret
```

### Redis Service Instance and Binding

```yaml
apiVersion: services.cloud.sap.com/v1
kind: ServiceInstance
metadata:
  name: mdm-redis
  namespace: mdm
spec:
  serviceOfferingName: redis-cache
  servicePlanName: standard
---
apiVersion: services.cloud.sap.com/v1
kind: ServiceBinding
metadata:
  name: mdm-redis-binding
  namespace: mdm
spec:
  serviceInstanceName: mdm-redis
  secretName: mdm-redis-secret
```

---

## MDM Core API Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mdm-core-api
  namespace: mdm
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mdm-core-api
  template:
    metadata:
      labels:
        app: mdm-core-api
    spec:
      containers:
      - name: mdm-core-api
        image: your-registry/mdm-core-api:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: AEM_HOST
          valueFrom:
            secretKeyRef: { name: mdm-aem-secret, key: uri }
        - name: AEM_MSG_VPN
          valueFrom:
            secretKeyRef: { name: mdm-aem-secret, key: msgVpn }
        - name: AEM_CLIENT_USERNAME
          valueFrom:
            secretKeyRef: { name: mdm-aem-secret, key: clientUsername }
        - name: AEM_CLIENT_PASSWORD
          valueFrom:
            secretKeyRef: { name: mdm-aem-secret, key: clientPassword }
        - name: XSUAA_JWKS_URI
          valueFrom:
            secretKeyRef: { name: mdm-xsuaa-secret, key: jwks_uri }
        - name: XSUAA_ISSUER_URI
          valueFrom:
            secretKeyRef: { name: mdm-xsuaa-secret, key: url }
        - name: MONGODB_URI
          valueFrom:
            secretKeyRef: { name: mdm-mongodb-secret, key: connectionString }
        - name: REDIS_HOST
          valueFrom:
            secretKeyRef: { name: mdm-redis-secret, key: host }
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef: { name: mdm-redis-secret, key: password }
        livenessProbe:
          httpGet: { path: /actuator/health/liveness, port: 8080 }
          initialDelaySeconds: 30
          failureThreshold: 3
          periodSeconds: 10
        readinessProbe:
          httpGet: { path: /actuator/health/readiness, port: 8080 }
          initialDelaySeconds: 20
          failureThreshold: 2
          periodSeconds: 5
        resources:
          requests: { memory: "512Mi", cpu: "250m" }
          limits:   { memory: "1Gi",   cpu: "1000m" }
```

---

## Kyma API Rule — JWT-Protected Endpoint

```yaml
apiVersion: gateway.kyma-project.io/v1beta1
kind: APIRule
metadata:
  name: mdm-core-api-rule
  namespace: mdm
spec:
  gateway: kyma-system/kyma-gateway
  host: mdm-api.your-cluster.kyma.ondemand.com
  service:
    name: mdm-core-api
    port: 8080
  rules:
  - path: /api/v1/.*
    methods: ["GET", "POST", "PUT", "DELETE"]
    accessStrategies:
    - handler: jwt
      config:
        jwksUrls:
        - ${XSUAA_JWKS_URI}
        trustedIssuers:
        - ${XSUAA_ISSUER_URI}
  - path: /actuator/health
    methods: ["GET"]
    accessStrategies:
    - handler: noop   # Health endpoint open, no auth required
```

---

## KEDA Autoscaler — Consumer Router

```yaml
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
  # Scale on Kafka consumer lag
  - type: kafka
    metadata:
      topic: mdm.customer.events
      consumerGroup: consumer-router-group
      bootstrapServers: redpanda.mdm.svc.cluster.local:9092
      lagThreshold: "1000"
  # Scale on AEM queue depth (Solace KEDA scaler)
  - type: solace-event-queue
    metadata:
      solaceSempBaseURL: ${AEM_SEMP_URL}
      messageVpn: ${AEM_MSG_VPN}
      queueName: mdm.consumer-router.queue
      messageCountTarget: "500"
      authenticationRef: solace-auth-secret
```

---

## Kyma Telemetry Pipelines

```yaml
# Metrics → Grafana Cloud / Prometheus remote write
apiVersion: telemetry.kyma-project.io/v1alpha1
kind: MetricPipeline
metadata:
  name: mdm-metrics
spec:
  output:
    otlp:
      endpoint:
        value: https://your-grafana-cloud-otlp-endpoint
      authentication:
        basic:
          user:
            valueFrom:
              secretKeyRef: { name: grafana-cloud-secret, key: username }
          password:
            valueFrom:
              secretKeyRef: { name: grafana-cloud-secret, key: password }
---
# Logs → SAP Cloud Logging
apiVersion: telemetry.kyma-project.io/v1alpha1
kind: LogPipeline
metadata:
  name: mdm-logs
spec:
  input:
    application:
      namespaces:
        include: ["mdm"]
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
      tls.disabled: false
```

---

## XSUAA Security Config (`xs-security.json`)

Defines OAuth2 scopes and role templates for MDM RBAC. Deployed as part of BTP service instance configuration.

```json
{
  "xsappname": "mdm-core-api",
  "tenant-mode": "shared",
  "scopes": [
    { "name": "$XSAPPNAME.mdm.read",           "description": "Read any MDM object" },
    { "name": "$XSAPPNAME.mdm.write.central",  "description": "Create/update centrally managed objects" },
    { "name": "$XSAPPNAME.mdm.write.local",    "description": "Create/update locally managed objects for own site" },
    { "name": "$XSAPPNAME.mdm.extend",         "description": "Add extension attributes to existing objects" },
    { "name": "$XSAPPNAME.mdm.admin",          "description": "Platform administration" }
  ],
  "role-templates": [
    {
      "name": "MDM_ReadOnly",
      "description": "Read access to all MDM golden records",
      "scope-references": ["$XSAPPNAME.mdm.read"]
    },
    {
      "name": "MDM_CentralSteward",
      "description": "Data Team steward — manages centrally governed objects",
      "scope-references": [
        "$XSAPPNAME.mdm.read",
        "$XSAPPNAME.mdm.write.central"
      ]
    },
    {
      "name": "MDM_SiteCoordinator",
      "description": "Site Operations — manages locally governed objects for own site",
      "scope-references": [
        "$XSAPPNAME.mdm.read",
        "$XSAPPNAME.mdm.write.local"
      ]
    },
    {
      "name": "MDM_SystemExtender",
      "description": "Satellite system service account — can add extension attributes",
      "scope-references": [
        "$XSAPPNAME.mdm.read",
        "$XSAPPNAME.mdm.extend"
      ]
    },
    {
      "name": "MDM_Admin",
      "description": "Platform administrator",
      "scope-references": [
        "$XSAPPNAME.mdm.read",
        "$XSAPPNAME.mdm.write.central",
        "$XSAPPNAME.mdm.write.local",
        "$XSAPPNAME.mdm.extend",
        "$XSAPPNAME.mdm.admin"
      ]
    }
  ]
}
```

---

## Prometheus Alert Rules

```yaml
groups:
- name: mdm-alerts
  rules:
  - alert: KafkaConsumerLagHigh
    expr: kafka_consumer_group_lag > 10000
    for: 5m
    labels: { severity: critical }
    annotations:
      summary: "Consumer {{ $labels.consumergroup }} lag is {{ $value }}"

  - alert: DebeziumConnectorDown
    expr: debezium_connector_status != 1
    for: 2m
    labels: { severity: critical }
    annotations:
      summary: "CDC pipeline broken — changes not flowing to AEM"

  - alert: MDMApiErrorRateHigh
    expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.05
    for: 3m
    labels: { severity: critical }

  - alert: RedisCacheHitRateLow
    expr: >
      redis_keyspace_hits_total /
      (redis_keyspace_hits_total + redis_keyspace_misses_total) < 0.7
    for: 10m
    labels: { severity: warning }

  - alert: ScheduledJobMissed
    expr: mdm_scheduler_last_run_seconds > 90000
    for: 0m
    labels: { severity: warning }
    annotations:
      summary: "Nightly delta job has not run in 25+ hours"
```
