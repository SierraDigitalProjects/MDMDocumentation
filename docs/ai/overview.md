# AI in MDM — Architecture and Monitoring

AI improves the Sierra Aura system in two distinct areas: **accelerating development** (reducing the cost and time to build features) and **augmenting autonomous operations** (moving beyond threshold-based alerting to intelligent error detection, root cause analysis, and self-healing).

---

## 1. AI in Architecture and Feature Development

### Where AI Accelerates MDM Build

| Development Task | Without AI | With AI (Claude API) | Effort Reduction |
|---|---|---|---|
| Object schema definition from source system | Manual field mapping workshops | AI ingests source system metadata, proposes MDM object model | ~60% |
| CPI iFlow Groovy transform scripts | Developer writes from scratch | AI generates from natural language description of mapping rules | ~70% |
| Spring Boot service stubs | Boilerplate from templates | AI generates full service, repository, and test class | ~65% |
| RBAC scope definitions (`xs-security.json`) | Manual design + trial/error | AI generates from object type + role description | ~50% |
| Prometheus alert rules | Manual PromQL authoring | AI generates rules from plain-English alert intent | ~55% |
| MongoDB index recommendations | DBA review of query patterns | AI analyses query shapes and recommends indexes | ~50% |
| Unit and integration test generation | Manual test authoring | AI generates test cases from service interface | ~70% |

### AI-Assisted Schema Definition

When onboarding a new master data object type, the AI agent:

1. Accepts source system metadata (S/4HANA field catalog, Salesforce object schema, Workday field list)
2. Proposes a normalised MDM `coreAttributes` model with field names, types, and constraints
3. Identifies duplicate or semantically equivalent fields across source systems
4. Suggests extension namespaces per satellite system

```python
# Example: AI-assisted schema generation via Claude API
import anthropic

client = anthropic.Anthropic()

source_metadata = """
S/4HANA Business Partner fields:
  BusinessPartner (Char 10), BusinessPartnerFullName (Char 81),
  BusinessPartnerCategory (Char 1), Country (Char 3), PostalCode (Char 10)

Salesforce Account fields:
  AccountId, Name, BillingCountry, BillingPostalCode, Type, Industry
"""

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=2048,
    messages=[{
        "role": "user",
        "content": f"""
        Given these source system fields, propose a normalised MDM Customer object model.
        Output as JSON with coreAttributes (canonical fields), extensions.S4HANA
        (SAP-specific fields), and extensions.CRM (CRM-specific fields).

        Source metadata:
        {source_metadata}
        """
    }]
)
print(response.content[0].text)
```

### AI-Generated CPI iFlow Transform Scripts

Instead of manually writing Groovy mapping scripts:

```python
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=4096,
    messages=[{
        "role": "user",
        "content": """
        Write a CPI Groovy script that transforms a SAP Business Partner OData payload
        into the MDM Customer JSON format. Rules:
        - Map BusinessPartnerFullName → coreAttributes.name
        - Map Country → coreAttributes.country (ISO 3166 alpha-2)
        - Map BusinessPartnerCategory: '1' → 'Person', '2' → 'Organisation'
        - Set metadata.leadSystem = 'S4HANA'
        - Preserve all address fields under coreAttributes
        """
    }]
)
```

Each generated script is reviewed by a developer before deployment to CPI — AI produces the first draft, human validates.

---

## 2. AI in Monitoring — Autonomous Operations

### Limitations of Threshold-Based Alerting

Traditional Prometheus alert rules are brittle:

- Rules fire when a threshold is crossed — they cannot explain **why**
- High alert volume during incidents creates noise that obscures the root cause
- Robusta runbooks are hardcoded — they can only fix what was anticipated at design time
- No cross-system correlation: a CPI iFlow failure caused by a MongoDB slowdown requires a human to connect the two

### AI Monitoring Architecture

```
Alert / Log / Metric Event
        │
        ▼
┌───────────────────────────────────────────────────────┐
│              AI Monitoring Agent (Claude API)          │
│                                                       │
│  [1] Retrieve context                                 │
│      - Recent logs from SAP Cloud Logging             │
│      - Active Prometheus alerts                       │
│      - AEM queue depth from SEMP API                  │
│      - CPI Error Inbox entries                        │
│      - Recent MongoDB / Redis metrics                 │
│                                                       │
│  [2] Analyse and correlate                            │
│      - Identify root cause across layers              │
│      - Determine blast radius (which consumers affected)│
│      - Classify: transient / systemic / data issue    │
│                                                       │
│  [3] Propose action                                   │
│      - Remediation steps ranked by confidence         │
│      - Estimated impact of each action                │
│                                                       │
│  [4] Execute (approved actions only)                  │
│      - Restart pod / flush cache / retry CPI iFlow    │
│      - Post enriched incident to Slack                │
└───────────────────────────────────────────────────────┘
```

### AI Use Cases in Monitoring

#### Error Detection — Log Analysis

Rather than pattern-matching log lines with regex rules, the AI reads structured log streams and identifies anomalies in context:

```python
def analyse_mdm_error(alert_context: dict) -> str:
    # Fetch recent logs from SAP Cloud Logging for the affected service
    logs = fetch_logs(service="mdm-core-api", last_minutes=15)
    aem_stats = fetch_aem_queue_stats()
    cpi_errors = fetch_cpi_error_inbox()

    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": f"""
            MDM system alert: {alert_context['summary']}

            Recent mdm-core-api logs (last 15 min):
            {logs}

            AEM queue stats:
            {aem_stats}

            CPI Error Inbox (last 10 entries):
            {cpi_errors}

            Analyse the root cause. Is this a:
            1. Transient infrastructure issue (retry will fix)
            2. Data issue (bad payload from lead system)
            3. Integration issue (CPI iFlow misconfiguration)
            4. Capacity issue (AEM queue backing up)

            Recommend the single most impactful remediation action.
            """
        }]
    )
    return response.content[0].text
```

#### Anomaly Diagnosis — Cross-Layer Correlation

The AI correlates signals that threshold alerting treats as separate incidents:

| Scenario | Traditional Alerting | AI Correlation |
|---|---|---|
| MongoDB slow → Redis miss spike → API latency up | 3 separate alerts, 3 separate Slack messages | Single diagnosis: "MongoDB degradation causing cache bypass — recommend scaling MongoDB or serving stale cache" |
| CPI iFlow failure → AEM DLQ growing → consumer not receiving data | 2 alerts, manual investigation | Single diagnosis: "CPI transform failure due to unexpected null field in S/4HANA payload — recommend adding null check at step 3" |
| Scheduler job missed + AEM queue growing | 2 unrelated alerts | Single diagnosis: "Nightly batch did not drain queue — AEM backlog now 48h of changes — recommend immediate manual trigger of delta job" |

#### Auto-Fix Recommendations

AI generates specific, contextual fix steps — not generic runbook instructions:

```
Alert: AemDlqMessagePresent — mdm.erp.vendor.scheduled.queue DLQ has 47 messages

AI Analysis:
Root cause: CPI iFlow MDM_Vendor_To_ERP_NightlyBatch failed on 47 messages.
Examining CPI error logs: "IDoc CREMAS05 segment E1LFM1M field LIFNR exceeds
10 characters — source vendor ID 'VEND-EXTERNAL-01234' is 18 chars."

Recommended fix (confidence: 94%):
1. Update CPI iFlow step 6 Groovy script to truncate LIFNR to 10 chars
   or map to a validated internal ERP vendor ID
2. Retry failed messages from CPI Error Inbox after fix is deployed
3. Add input validation in MDM Consumer Registry to reject vendor IDs > 10 chars

Estimated time to resolve: 30 minutes
Impact if unresolved: ERP vendor master will not receive 47 vendor updates
```

#### Self-Healing Loop

For pre-approved, low-risk actions, the AI agent executes without waiting for human approval:

```python
APPROVED_AUTO_ACTIONS = [
    "restart_pod",           # pod crash-loop with known pattern
    "flush_redis_cache",     # cache inconsistency confirmed by AI
    "retry_cpi_messages",    # transient 5xx, payload unchanged
]

def self_healing_loop(alert: dict):
    analysis = analyse_mdm_error(alert)
    action = extract_recommended_action(analysis)

    if action["type"] in APPROVED_AUTO_ACTIONS and action["confidence"] > 0.90:
        execute_action(action)
        notify_slack(f"Auto-resolved: {action['description']}\n\nAI analysis: {analysis}")
    else:
        # Escalate to human with full AI context
        notify_pagerduty(alert, ai_context=analysis)
```

---

## 3. Cost Economics — Token Model

Pricing based on Claude API (claude-opus-4-6) as of 2025: **$15 / 1M input tokens, $75 / 1M output tokens**.

### Development Phase — Feature Generation Costs

| AI Task | Est. Input Tokens | Est. Output Tokens | Cost per Task | Tasks per Project |
|---|---|---|---|---|
| Schema definition (one object type) | 2,000 | 1,500 | ~$0.14 | 20 → **$2.80** |
| CPI iFlow Groovy script | 1,500 | 3,000 | ~$0.25 | 15 → **$3.75** |
| Spring Boot service stub | 1,000 | 4,000 | ~$0.32 | 10 → **$3.20** |
| Unit test generation (per service) | 2,000 | 5,000 | ~$0.41 | 10 → **$4.10** |
| Prometheus alert rules (full set) | 1,000 | 2,000 | ~$0.18 | 1 → **$0.18** |
| xs-security.json RBAC definition | 800 | 1,200 | ~$0.10 | 1 → **$0.10** |
| **Total AI development cost** | | | | **~$14** |

!!! success "Development ROI"
    Generating ~$14 of API calls replaces an estimated 8–12 person-days of boilerplate authoring. At a fully-loaded developer rate of $800/day that is **$6,400–$9,600 of saved effort** for $14 of API cost.

### Operations Phase — Monitoring Cost per Event

| AI Monitoring Task | Est. Input Tokens | Est. Output Tokens | Cost per Event |
|---|---|---|---|
| Error log analysis (15 min logs + context) | 8,000 | 800 | ~$0.18 |
| Anomaly diagnosis (metrics + logs) | 6,000 | 1,000 | ~$0.17 |
| Root cause + fix recommendation | 10,000 | 1,500 | ~$0.26 |
| Self-healing decision (low-risk action) | 4,000 | 400 | ~$0.09 |

#### Monthly Operations Cost Model

| Scenario | Events/day | AI calls/event | Cost/month |
|---|---|---|---|
| **Low activity** — stable system, few alerts | 5 | 2 | ~$1.60 |
| **Normal operations** — typical alert volume | 20 | 2 | ~$6.40 |
| **Incident day** — major degradation | 100 | 3 | ~$24 per incident |
| **Full autonomous ops** — all log anomalies analysed | 500 | 1 | ~$80/month |

!!! info "Cost vs Value"
    At $80/month for full autonomous log analysis, a single incident that AI diagnoses in 5 minutes instead of a developer spending 2 hours represents $250+ of saved on-call time — the monthly AI cost is recovered in one incident.

### Token Optimisation Strategies

| Strategy | Token Savings | Trade-off |
|---|---|---|
| Use `claude-haiku-4-5` for high-volume, low-complexity log triage | ~90% cost reduction | Lower reasoning quality for complex incidents |
| Pre-filter logs before sending to AI (send only ERROR/WARN lines) | 60–80% input reduction | May miss context from INFO logs |
| Cache AI analysis for identical alert signatures | Skip duplicate calls | Stale analysis if system state changed |
| Limit log window to 5 min (not 15) for transient alerts | ~60% input reduction | May miss slow-developing issues |
| Use `claude-sonnet-4-6` for root cause, `haiku-4-5` for triage routing | ~70% cost reduction | Two-tier pipeline adds latency |

**Recommended tiered model:**

```
Alert fires
    │
    ▼ Haiku-4.5 triage ($0.02/call)
    ├── "Transient / known pattern" → auto-retry, no escalation
    ├── "Unknown / complex"        → Opus-4.6 deep analysis ($0.26/call)
    └── "Critical / data loss"     → Opus-4.6 + immediate page
```

---

## 4. Build vs Buy

| Approach | When to Use | Indicative Cost |
|---|---|---|
| **Claude API directly** (as above) | Full control, MDM-specific context, custom self-healing logic | ~$80–200/month ops |
| **Grafana Sift** (AI in Grafana Cloud) | Already on Grafana Cloud, want AI in existing dashboards | Grafana Cloud pricing |
| **Dynatrace Davis AI** | Large enterprise, broad APM needed beyond MDM, budget available | $$$$ |
| **Azure Monitor AI** (if on AKS) | AKS deployment, Azure ecosystem | Azure pricing |
| **SAP AI Core** (if on BTP) | BTP-native, SAP ecosystem, want managed LLM on BTP | BTP credits |

**Recommendation for Sierra Aura on BTP Kyma:** Use Claude API directly for the first 12 months. The MDM context (AEM queues, CPI iFlows, MongoDB Change Streams) is specific enough that pre-built AI monitoring platforms will require significant configuration to match. Direct API use gives full control at minimal cost, and the prompts built during this phase become reusable assets.

At scale (>10 sites, >50 object types), evaluate **SAP AI Core** for BTP-native hosting of the AI monitoring agent — it removes external API dependency and can be bound as a BTP service instance to the Kyma deployment.
