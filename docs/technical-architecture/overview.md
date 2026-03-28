# Technical Architecture Overview

## Architecture Pattern

The SierraDock MDM solution follows a **microservices + event-driven architecture**. Every master data change is propagated to satellite systems via an event broker, with configurable delivery schedules per consumer and field-level filtering to avoid unnecessary fan-out.

```
Lead Systems (S4HANA, CRM, Workday, etc.)
        │
        ▼ REST / IDoc inbound (via CPI)
┌───────────────────────────────────────────────┐
│              MDM Core API (Spring Boot)        │
│  ┌──────────┐ ┌──────────┐ ┌───────────────┐  │
│  │  Schema  │ │   RBAC   │ │ Change Tracker│  │
│  │ Registry │ │ XSUAA/KC │ │  (field diff) │  │
│  └──────────┘ └──────────┘ └───────────────┘  │
└──────────────────┬────────────────────────────┘
                   │ Write                 │ Read
                   ▼                       ▼
              MongoDB ◄──── CDC         Redis (Read-through)
              (Change                   (RedisJSON + RediSearch)
               Streams)
                   │
                   ▼
         ┌─────────────────────┐
         │   Event Broker      │
         │  AEM (Solace) or    │
         │  Kafka / Redpanda   │
         └──────────┬──────────┘
                    │
         ┌──────────▼──────────┐
         │  CPI iFlows         │
         │  (field filter +    │
         │   transform +       │
         │   schedule)         │
         └──────────┬──────────┘
                    │
          ┌─────────┴──────────┐
          ▼                    ▼
    Satellite System A   Satellite System B
    (real-time push)     (nightly batch IDoc)
```

---

## Core Capabilities

| Capability | Implementation |
|---|---|
| Custom object model definition | MongoDB dynamic schema + Schema Definition Service |
| RBAC per master data object | SAP XSUAA (BTP) or Keycloak (AKS) + Spring Security |
| Lead system inbound | REST API + CPI iFlows (IDoc / OData adapters) |
| Field-level change tracking | Application-level diff + MongoDB Change Streams |
| Consumer field-interest registration | Consumer Registry Service + CPI field filter script |
| Event fan-out | SAP AEM (Solace) or Strimzi Kafka / Redpanda |
| Configurable delivery schedule | Quartz Scheduler + CPI Timer iFlow per consumer |
| Read-through cache | Redis Stack (RedisJSON + RediSearch) |
| Attribute extension per system | `extensions.{SYSTEM_ID}` map in MongoDB document |
| Audit trail | Immutable `change_records` MongoDB collection |

---

## Full Stack Summary

| Layer | Recommended (BTP Kyma) | Alternative (AKS) |
|---|---|---|
| **Compute** | SAP BTP Kyma (managed K8s) | Azure AKS |
| **Backend services** | Java Spring Boot 3.x | Java Spring Boot 3.x |
| **Database** | MongoDB (Community Operator on Kyma) | MongoDB Atlas on Azure |
| **Cache** | Redis (BTP Hyperscaler Option Service) | Redis Stack on AKS |
| **Event broker** | SAP Advanced Event Mesh (AEM / Solace) | Strimzi Kafka or Redpanda |
| **CDC** | MongoDB Change Streams + Debezium | MongoDB Change Streams + Debezium |
| **Integration middleware** | SAP Cloud Integration (CPI) | Custom adapters or MuleSoft |
| **IAM / RBAC** | SAP IAS + XSUAA | Keycloak |
| **API Gateway** | Kyma API Gateway (Ory + Istio) | Kong (self-managed) |
| **Scheduler** | Quartz (clustered, MongoDB-persisted) | Quartz (clustered) |
| **Schema Registry** | Apicurio Registry (on Kyma) | Apicurio Registry |
| **Service mesh** | Istio (Kyma built-in) | Istio |
| **Observability** | Kyma Telemetry + SAP Cloud Logging | OpenTelemetry + Loki + Tempo |
| **Auto-scaling** | KEDA (Kyma built-in) | KEDA |

---

## Microservice Breakdown

| Service | Responsibility |
|---|---|
| **MDM Core API** | Create / update / read master data objects, RBAC enforcement, change tracking, AEM publish |
| **Schema Definition Service** | Define and version custom object types and their field schemas |
| **Consumer Registry Service** | Store and manage consumer registrations, field interest lists, delivery modes |
| **Fanout Router** | Match changedFields against consumer interest registrations, route to appropriate queues |
| **Scheduler Service** | Manage per-object Quartz jobs, sync schedule config to CPI Timer iFlows |
| **Read Cache Service** | Redis read-through layer, cache invalidation listener |
| **Audit Service** | Query and export immutable change records for compliance |

---

## Key Architecture Decisions

!!! info "Event Broker Choice"
    Use **SAP Advanced Event Mesh (AEM)** if majority of consumers are SAP systems — CPI pre-built adapters eliminate custom integration code. Use **Redpanda** (Kafka-compatible) for non-SAP-heavy landscapes or when open-standard tooling is preferred.

!!! info "Database Choice"
    **MongoDB** is the correct choice for this use case — dynamic schema supports evolving custom object definitions, Change Streams provide built-in CDC, and the aggregation pipeline handles field-level diff efficiently.

!!! warning "Kyma Eventing Limitation"
    Kyma's built-in NATS-based eventing is **not sufficient** for MDM fan-out. It lacks compacted topics (last-value per object key) and does not support scheduled/batched delivery. Use AEM or Kafka for external consumer fan-out; reserve Kyma Eventing for internal service-to-service events only.
