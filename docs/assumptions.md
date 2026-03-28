# Assumptions

These assumptions underpin the architecture, requirements, and cost estimates in this document. If any assumption is invalidated, the impacted sections should be revisited.

---

## Business Assumptions

| # | Assumption | Impact if Wrong |
|---|---|---|
| BA-01 | The Data Team will own and resource the central steward role for centrally managed objects | Without a staffed steward, approval workflows stall and golden records cannot be published |
| BA-02 | Site Operations teams will nominate and resource Site Data Coordinators per site | Without coordinators, locally managed objects cannot be maintained in MDM |
| BA-03 | Business sponsors have approved MDM as the system of record for the 31 non-SAP objects | Without sponsorship, MDM cannot displace existing (informal) processes — objects remain ungoverned |
| BA-04 | Data governance policies (retention, quality thresholds, approval SLAs) will be defined before Phase 1 go-live | Without policies, the platform can store data but cannot enforce governance |
| BA-05 | CRM, EH&S, and HR domain leads will participate in data migration workshops to provide initial load data | Without their input, initial data quality for non-SAP objects will be low |
| BA-06 | All satellite systems will migrate to consuming master data from MDM (via API or AEM) within 24 months | If systems continue reading directly from S/4HANA, the MDM golden record has no authority |

---

## Technical Assumptions

| # | Assumption | Impact if Wrong |
|---|---|---|
| TA-01 | SAP S/4HANA is the current system of record for all 52 SAP-managed objects listed in the inventory | If any object has a different source (MDG, legacy ECC), inbound integration design changes |
| TA-02 | SAP Cloud Integration (CPI) is available and licensed within the BTP environment | CPI removal requires replacing all iFlows with custom integration code (~+40% effort) |
| TA-03 | SAP Advanced Event Mesh (AEM) is available and licensed within the BTP environment | AEM removal requires deploying Strimzi Kafka or Redpanda on Kyma |
| TA-04 | BTP Kyma Runtime is available and sized to support the MDM workload (minimum 4 nodes, 32GB RAM) | Undersized Kyma requires AKS migration — significant rework |
| TA-05 | The S/4HANA Business Partner unified model is fully activated (not split Customer/Vendor) | Split Customer/Vendor requires separate IDoc types and separate MDM object models for BP |
| TA-06 | SAP Cloud Connector is installed and configured for on-premise S/4HANA connectivity to BTP | Without Cloud Connector, inbound integration from on-premise S/4HANA requires VPN redesign |
| TA-07 | MongoDB Community Operator can be deployed on the Kyma cluster (or MongoDB Atlas is available via BTP marketplace) | If neither is available, database must be self-hosted outside Kyma — adds ops overhead |
| TA-08 | An enterprise Identity Provider (IdP) — Azure AD or SAP IAS — is available for SSO federation | Without IdP, user management must be handled locally in XSUAA — significant overhead |
| TA-09 | Network latency between Kyma cluster and on-premise S/4HANA is < 100ms | Higher latency affects inbound synchronous API calls from CPI |
| TA-10 | The CPI iFlow development model (one iFlow per satellite system per object type) is acceptable to the integration team | If a different integration pattern is mandated, iFlow designs need to be revisited |

---

## Data Assumptions

| # | Assumption | Impact if Wrong |
|---|---|---|
| DA-01 | Source system data quality is sufficient for initial migration (< 20% records requiring manual cleansing) | Higher cleansing rates push out Phase 1 go-live and increase migration effort |
| DA-02 | Record volumes are within the estimates below | Higher volumes require MongoDB sharding or Kyma node scaling earlier than planned |
| DA-03 | The 83 objects in the inventory are the full scope — no undiscovered objects will be added mid-project | New objects discovered late in the project require schema definition and integration rework |
| DA-04 | Duplicate records across source systems are addressable with deterministic matching keys (SAP BP number, Material number, Employee ID) | If no reliable match key exists, probabilistic matching adds significant complexity to deduplication |

### Estimated Record Volumes

| Object Type | Estimated Records | Growth Rate |
|---|---|---|
| Business Partner (all) | 50,000–200,000 | Low |
| Material Master | 100,000–500,000 | Medium |
| Equipment | 20,000–100,000 | Low |
| Customer / Vendor | 30,000–150,000 | Low |
| Pricing Condition Records | 500,000–2,000,000 | High |
| HR Employees | 1,000–10,000 | Low |

---

## Organisational Assumptions

| # | Assumption | Impact if Wrong |
|---|---|---|
| OA-01 | The MDM programme has executive sponsorship and a named programme lead | Without sponsorship, cross-team data governance decisions will stall |
| OA-02 | The integration team has at least one developer experienced in CPI iFlow development | Without CPI experience, iFlow development time approximately doubles |
| OA-03 | A BTP administrator is available to provision services, manage service bindings, and configure Kyma | Without a BTP admin, platform setup and incident recovery is blocked |
| OA-04 | The project will follow phased delivery — Phase 1 scope is not expanded mid-flight | Scope creep in Phase 1 pushes the foundation delivery and delays Phase 2 fan-out |
