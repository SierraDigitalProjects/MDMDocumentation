# Non-Functional Requirements

---

## NFR-01 — Performance

| ID | Requirement | Target | Notes |
|---|---|---|---|
| NFR-01.1 | MDM API response time for single record lookup | < 500ms p95 | Under normal load |
| NFR-01.2 | MDM API response time for search queries | < 2s p95 | Up to 1000 results |
| NFR-01.3 | Inbound replication processing time (SAP → MDM golden record) | < 5 minutes | For critical objects |
| NFR-01.4 | Outbound publish time (MDM → downstream) after approval | < 2 minutes | |
| NFR-01.5 | Bulk load throughput for initial migration | > 10,000 records/hour per object type | |
| NFR-01.6 | UI page load time | < 3s | Steward workbench pages |

---

## NFR-02 — Availability and Reliability

| ID | Requirement | Target | Notes |
|---|---|---|---|
| NFR-02.1 | MDM platform uptime (excluding planned maintenance) | 99.5% | ~44 hours downtime/year |
| NFR-02.2 | Planned maintenance windows must not fall during business hours | Nights/weekends only | Business hours TBD |
| NFR-02.3 | Integration message queue must be durable — no message loss on platform restart | Zero message loss | Dead letter queue required |
| NFR-02.4 | MDM must support active/passive failover | RPO < 1 hour, RTO < 4 hours | |
| NFR-02.5 | SAP replication must resume automatically after CPI outage without data loss | Automatic retry | |

---

## NFR-03 — Scalability

| ID | Requirement | Target |
|---|---|---|
| NFR-03.1 | MDM must support the full 83-object inventory without platform reconfiguration | All 83 objects |
| NFR-03.2 | MDM must scale to accommodate new sites without architectural changes | Horizontal site scaling |
| NFR-03.3 | MDM must support a minimum of 1 million golden records per major object type (Material Master, Business Partner) | 1M+ records |
| NFR-03.4 | MDM must support concurrent access by all site users simultaneously without degradation | All users concurrent |

---

## NFR-04 — Security

| ID | Requirement | Notes |
|---|---|---|
| NFR-04.1 | All data in transit must be encrypted (TLS 1.2 minimum, TLS 1.3 preferred) | Applies to UI, APIs, and integration messages |
| NFR-04.2 | All data at rest must be encrypted | AES-256 or equivalent |
| NFR-04.3 | MDM must support SSO via SAML 2.0 or OIDC against enterprise IdP | No local password management |
| NFR-04.4 | API access must use OAuth 2.0 with client credentials for service accounts | No API key authentication |
| NFR-04.5 | MDM must log all authentication events (success and failure) | Retained per audit requirements |
| NFR-04.6 | Sensitive attributes (e.g., tax IDs, bank data) must be masked in UI for non-privileged roles | Field-level masking |
| NFR-04.7 | MDM must pass annual security assessment | Penetration test + vulnerability scan |

---

## NFR-05 — Integration Compatibility

| ID | Requirement | Notes |
|---|---|---|
| NFR-05.1 | MDM must be compatible with SAP Integration Suite (CPI) as the integration middleware | No direct RFC/BAPI calls from MDM |
| NFR-05.2 | MDM must expose REST/OData APIs for consumer integration | JSON format minimum |
| NFR-05.3 | MDM must support IDoc inbound processing (via CPI adapter) | For SAP replication |
| NFR-05.4 | MDM must support webhook-based event publishing for downstream notification | Push model preferred over polling |
| NFR-05.5 | MDM APIs must be versioned to prevent breaking changes for consumers | Semantic versioning |

---

## NFR-06 — Operability

| ID | Requirement | Notes |
|---|---|---|
| NFR-06.1 | MDM must provide an operations dashboard showing replication health, queue depth, and error rates | Real-time |
| NFR-06.2 | Failed integration messages must be reprocessable from the operations dashboard without developer involvement | Self-service ops |
| NFR-06.3 | MDM must support export of any golden record set in CSV and JSON formats | For reporting and migration |
| NFR-06.4 | MDM deployment must support infrastructure-as-code for repeatable environment provisioning | CI/CD compatible |
| NFR-06.5 | MDM must provide a sandbox/test environment that mirrors production configuration | Non-production parity |

---

## NFR-07 — Compliance

| ID | Requirement | Notes |
|---|---|---|
| NFR-07.1 | MDM must retain all record history for a minimum of 7 years | Regulatory minimum |
| NFR-07.2 | MDM must support data subject access requests (DSAR) — locating all records associated with an individual | GDPR / CCPA |
| NFR-07.3 | MDM must support right-to-erasure for non-regulated personal data | GDPR / CCPA |
| NFR-07.4 | EH&S objects (Hazardous Materials, MSDS, DOT) must support regulatory approval workflow with e-signature capture | EH&S compliance |
