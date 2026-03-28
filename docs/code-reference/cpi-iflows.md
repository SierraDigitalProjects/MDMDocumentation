# CPI iFlow Designs and Groovy Scripts

## iFlow Inventory

| iFlow Name | Direction | Pattern | Trigger |
|---|---|---|---|
| `S4_BusinessPartner_To_MDM_Customer` | S/4HANA → MDM | Inbound REST | IDoc / OData push from S4 |
| `MDM_Customer_To_CRM_Realtime` | MDM → CRM | Outbound real-time | AEM queue event |
| `MDM_Vendor_To_ERP_NightlyBatch` | MDM → ERP | Outbound scheduled | Timer (02:00 daily) |

---

## Inbound: S/4HANA Business Partner → MDM

```
┌──────────────────────────────────────────────────────────────┐
│  iFlow: S4_BusinessPartner_To_MDM_Customer                   │
│                                                              │
│  [1] SOAP/IDoc Receiver                                      │
│      S4HANA pushes DEBMAS IDoc or OData BusinessPartner      │
│       │                                                      │
│  [2] Message Validator                                       │
│      Validate mandatory fields present                       │
│       │                                                      │
│  [3] Groovy: Extract & Transform                             │
│      Map BP fields → MDM Customer JSON                       │
│       │                                                      │
│  [4] Request Reply: GET /api/v1/objects/{bpNumber}           │
│      Check if object already exists (create vs update)       │
│       │                                                      │
│  [5] Router: exists?                                         │
│      ├── YES → [6a] PUT /api/v1/objects/{id}                 │
│      └── NO  → [6b] POST /api/v1/objects                     │
│       │                                                      │
│  [7] Exception Handler                                       │
│      On error → route to AEM dead-letter queue               │
│       │                                                      │
│  [8] Success Logger                                          │
└──────────────────────────────────────────────────────────────┘
```

### Step 3 — S/4HANA BP → MDM Customer Transform

```groovy
import groovy.json.JsonSlurper
import groovy.json.JsonOutput

def Message processData(Message message) {
    def bp = new JsonSlurper().parseText(message.getBody(String))

    def mdmCustomer = [
        objectType: "Customer",
        coreAttributes: [
            businessPartnerId: bp.BusinessPartner,
            name             : bp.BusinessPartnerFullName,
            category         : bp.BusinessPartnerCategory, // 1=Person, 2=Org
            country          : bp.to_BusinessPartnerAddress?.results?.getAt(0)?.Country,
            street           : bp.to_BusinessPartnerAddress?.results?.getAt(0)?.StreetName,
            city             : bp.to_BusinessPartnerAddress?.results?.getAt(0)?.CityName,
            postalCode       : bp.to_BusinessPartnerAddress?.results?.getAt(0)?.PostalCode,
            language         : bp.Language,
            searchTerm       : bp.SearchTerm1
        ],
        metadata: [
            leadSystem: "S4HANA"
        ]
    ]

    message.setBody(JsonOutput.toJson(mdmCustomer))
    message.setHeader("Content-Type", "application/json")
    return message
}
```

### XSUAA Token Fetch (Groovy — reuse across all iFlows)

```groovy
def Message fetchToken(Message message) {
    def tokenUrl    = message.getProperties().get("XSUAA_TOKEN_URL")
    def clientId    = message.getProperties().get("MDM_CLIENT_ID")
    def clientSecret = message.getProperties().get("MDM_CLIENT_SECRET")

    def http = new URL(tokenUrl).openConnection()
    http.requestMethod = "POST"
    http.setRequestProperty("Content-Type", "application/x-www-form-urlencoded")
    http.doOutput = true
    http.outputStream.write(
        "grant_type=client_credentials&client_id=${clientId}&client_secret=${clientSecret}".bytes
    )

    def response = new groovy.json.JsonSlurper().parse(http.inputStream)
    message.setProperty("MDM_ACCESS_TOKEN", "Bearer ${response.access_token}")
    return message
}
```

---

## Outbound Real-Time: MDM → CRM

```
┌──────────────────────────────────────────────────────────────┐
│  iFlow: MDM_Customer_To_CRM_Realtime                         │
│                                                              │
│  [1] AMQP Receiver                                           │
│      Queue: mdm.crm.customer.realtime.queue                  │
│       │                                                      │
│  [2] Groovy: Field Interest Filter                           │
│      If no CRM-relevant fields changed → SKIP                │
│       │                                                      │
│  [3] Router: SKIP?                                           │
│      ├── YES → End (discard)                                 │
│      └── NO  ↓                                               │
│  [4] Groovy: Transform MDM → CRM format                      │
│       │                                                      │
│  [5] HTTP Receiver → CRM REST API                            │
│      POST /crm/api/accounts  (OAuth2 client credentials)     │
│       │                                                      │
│  [6] Exception Handler                                       │
│      Retry 3x exponential backoff → AEM DLQ after max retry  │
│       │                                                      │
│  [7] Acknowledge AMQP message                                │
└──────────────────────────────────────────────────────────────┘
```

### Step 2 — Field Interest Filter

```groovy
def Message filterByFieldInterest(Message message) {
    def body = new groovy.json.JsonSlurper().parseText(message.getBody(String))
    def crmInterestedFields = [
        "coreAttributes.name",
        "coreAttributes.country",
        "coreAttributes.email",
        "extensions.CRM.tier",
        "extensions.CRM.accountManager"
    ]

    def changedFields = body.changedFields as List<String>

    // Always deliver on create (changedFields is null)
    if (changedFields == null) {
        message.setProperty("SKIP", false)
        return message
    }

    def hasRelevantChange = changedFields.any { cf ->
        crmInterestedFields.any { interest -> cf == interest || cf.startsWith(interest) }
    }

    message.setProperty("SKIP", !hasRelevantChange)
    return message
}
```

### Step 4 — MDM → CRM Format Transform

```groovy
def Message transformToCrm(Message message) {
    def mdm  = new groovy.json.JsonSlurper().parseText(message.getBody(String))
    def core = mdm.payload.coreAttributes
    def ext  = mdm.payload.extensions?.CRM ?: [:]

    def crmAccount = [
        externalId   : mdm.objectId,
        mdmVersion   : mdm.version,
        accountName  : core.name,
        country      : core.country,
        accountTier  : ext.tier ?: "Standard",
        accountOwner : ext.accountManager,
        lastSyncedAt : mdm.occurredAt
    ]

    message.setBody(new groovy.json.JsonOutput().toJson(crmAccount))
    message.setHeader("Content-Type", "application/json")
    message.setHeader("X-MDM-Event-Id", mdm.eventId)
    message.setHeader("X-MDM-Correlation-Id", mdm.correlationId)
    return message
}
```

---

## Outbound Scheduled Batch: MDM → ERP

```
┌──────────────────────────────────────────────────────────────┐
│  iFlow: MDM_Vendor_To_ERP_NightlyBatch                       │
│                                                              │
│  [1] Timer Start Event                                       │
│      Cron: 0 2 * * *  (02:00 daily, configurable)           │
│       │                                                      │
│  [2] AMQP Receiver — Drain Queue                             │
│      Queue: mdm.erp.vendor.scheduled.queue                   │
│      Max: 10,000 messages per run                            │
│      Timeout: 30s (stop when queue empty)                    │
│       │                                                      │
│  [3] Aggregator                                              │
│      Collect all messages into one batch                     │
│       │                                                      │
│  [4] Groovy: Deduplicate by objectId                         │
│      Keep only latest version per vendor ID                  │
│       │                                                      │
│  [5] Splitter                                                │
│      Split batch into chunks of 100                          │
│       │                                                      │
│  [6] Groovy: Transform MDM → IDoc CREMAS format              │
│       │                                                      │
│  [7] IDoc Receiver → ERP via BTP Connectivity                │
│      Adapter: IDoc  MessageType: CREMAS05                    │
│       │                                                      │
│  [8] Exception Handler                                       │
│      Failed chunk → AEM DLQ + SAP Alert Notification         │
│       │                                                      │
│  [9] Batch Summary Logger                                    │
└──────────────────────────────────────────────────────────────┘
```

### Step 4 — Deduplicate by Object ID

```groovy
import groovy.json.JsonSlurper
import groovy.json.JsonOutput

def Message deduplicateByObjectId(Message message) {
    def messages = new JsonSlurper().parseText(message.getBody(String)) as List

    // Keep only the highest version per objectId
    def deduped = messages
        .groupBy { it.objectId }
        .collect { id, versions -> versions.max { it.version } }

    message.setProperty("DEDUP_SKIPPED", messages.size() - deduped.size())
    message.setBody(JsonOutput.toJson(deduped))
    return message
}
```

### Step 6 — MDM → CREMAS IDoc Transform

```groovy
def Message transformToCremas(Message message) {
    def vendors = new groovy.json.JsonSlurper().parseText(message.getBody(String)) as List

    def writer = new StringWriter()
    def idoc   = new groovy.xml.MarkupBuilder(writer)

    idoc.CREMAS05 {
        IDOC(BEGIN: "1") {
            vendors.each { v ->
                def core = v.payload.coreAttributes
                E1LFM1M {
                    MSGFN("005")         // Change function
                    LIFNR(core.vendorId)
                    NAME1(core.name)
                    LAND1(core.country)
                    ORT01(core.city)
                    STRAS(core.street)
                    PSTLZ(core.postalCode)
                }
            }
        }
    }

    message.setBody(writer.toString())
    message.setHeader("Content-Type", "application/xml")
    return message
}
```

---

## CPI Secure Parameter Configuration

All credentials (MDM API URL, XSUAA token URL, client ID/secret, AEM connection) are stored as CPI Secure Parameters — never hardcoded in Groovy scripts.

| Parameter Name | Value Source |
|---|---|
| `MDM_API_URL` | Kyma API Gateway host |
| `XSUAA_TOKEN_URL` | XSUAA service binding secret |
| `MDM_CLIENT_ID` | XSUAA service binding |
| `MDM_CLIENT_SECRET` | XSUAA service binding |
| `AEM_HOST` | AEM service binding |
| `AEM_MSG_VPN` | AEM service binding |
| `AEM_CLIENT_USERNAME` | AEM service binding |
| `AEM_CLIENT_PASSWORD` | AEM service binding |
