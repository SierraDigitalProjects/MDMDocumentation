# Project Structure

This page describes how the documentation site is organised — where to find content, what each section covers, and how to add new pages or sections.

---

## Folder Layout

```
SierraMDM/
├── README.md                               # Setup and launch instructions
└── Documentation/
    ├── mkdocs.yml                          # Site configuration and navigation
    └── docs/
        ├── index.md                        # Home page / executive summary
        ├── goals-objectives.md             # Strategic goals and delivery phases
        ├── benefits.md                     # Business, technical, and cost benefits
        ├── assumptions.md                  # Project assumptions and risks
        ├── project-structure.md            # This page
        ├── inventory/                      # Master data object catalogue
        │   ├── overview.md                 # 83-object inventory summary and tier counts
        │   ├── centrally-managed.md        # SAP-managed, Data Team governed
        │   ├── locally-managed.md          # SAP-managed, site-scoped
        │   └── not-in-sap.md              # MDM as lead system (CRM, EH&S, HR)
        ├── requirements/                   # Functional and non-functional requirements
        │   ├── functional.md               # FR-01 through FR-09 (incl. enrichment governance)
        │   ├── non-functional.md           # NFR-01 through NFR-07
        │   └── open-questions.md           # Outstanding decisions and owners
        ├── architecture/                   # MDM architecture decisions and governance
        │   ├── lead-system-strategy.md     # Three-tier lead system model
        │   ├── integration-patterns.md     # CPI iFlow patterns and AEM topic design
        │   ├── rbac-governance.md          # XSUAA scopes, roles, and site isolation
        │   └── data-enrichment-governance.md  # Enrichment types, namespace model, workflows
        ├── technical-architecture/         # Platform and infrastructure design
        │   ├── overview.md                 # Full stack diagram and component responsibilities
        │   ├── compute-options.md          # BTP Kyma vs AKS comparison
        │   ├── event-broker.md             # AEM vs Kafka, effort reduction, recommendation
        │   └── data-layer.md               # MongoDB schema design, Redis caching, Change Streams
        ├── operations/                     # Ongoing platform operations
        │   ├── monitoring.md               # Autonomous monitoring — alerts, KEDA, Robusta runbooks
        │   └── cost.md                     # Infrastructure cost model and people costs
        ├── ai/
        │   └── overview.md                 # AI in architecture and monitoring, token cost model
        └── code-reference/                 # Implementation guides and code patterns
            ├── spring-boot.md              # MDM Core API — Spring Boot 3.x patterns
            ├── cpi-iflows.md               # CPI iFlow structure and Groovy scripts
            └── kubernetes.md               # Kyma manifests, KEDA scalers, Robusta runbooks
```

---

## Sections

| Section | Folder / File | Purpose | Audience |
|---|---|---|---|
| **Home** | `index.md` | Executive summary, 3-tier model, modules in scope, document owners | All stakeholders |
| **Goals & Objectives** | `goals-objectives.md` | Strategic goals, business and technical objectives (BO/TO), 5-phase delivery plan, success criteria | Programme Sponsors, MDM Architect |
| **Benefits** | `benefits.md` | Business, technical, and cost benefits; integration cost reduction; risk reduction | Programme Sponsors, Finance |
| **Assumptions** | `assumptions.md` | Business, technical, data, and organisational assumptions with impact-if-wrong analysis | All stakeholders |
| **Scope — Master Data Inventory** | `inventory/` | Full catalogue of all 83 master data objects classified by management tier and SAP module | Data Team, Functional Leads |
| **Requirements** | `requirements/` | Functional requirements (FR-01–FR-09), non-functional requirements (NFR-01–NFR-07), and open questions log | MDM Architect, Business Sponsors |
| **MDM Architecture** | `architecture/` | Lead system strategy, CPI integration patterns, RBAC and governance, data enrichment governance | MDM Architect, Integration Team, Data Governance |
| **Technical Architecture** | `technical-architecture/` | Platform stack, compute options, event broker selection, data layer design | MDM Engineer, Platform Architect |
| **Operations** | `operations/` | Autonomous monitoring configuration, alert rules, auto-remediation, and full cost of operations model | MDM Platform Engineer, Programme Sponsors |
| **AI in MDM** | `ai/overview.md` | How Claude API is used in architecture assistance and autonomous monitoring; token cost economics | MDM Architect, MDM Platform Engineer |
| **Code Reference** | `code-reference/` | Implementation patterns and example code for Spring Boot, CPI iFlows, and Kyma manifests | MDM Engineer, Integration Developer |
| **About** | `project-structure.md` | Documentation site layout and contribution guide | Contributors |

---

## Reader Journey

The documentation is ordered to support a natural reading flow from business context through to implementation:

```
Goals & Objectives    →  why we are building this and what success looks like
Benefits              →  quantified value and risk reduction
Assumptions           →  what must hold true for the plan to work
Scope                 →  which master data objects are in scope and how they are classified
Requirements          →  what the system must do (functional + non-functional)
Architecture          →  how the system is designed (MDM model + technical stack)
Operations            →  how it is run and what it costs
AI in MDM             →  where AI improves build speed and operational efficiency
Code Reference        →  how to implement it
```

---

## Modules (SAP Functional Areas)

The inventory and architecture sections are organised around these SAP functional modules:

| Module | Code | Coverage |
|---|---|---|
| Customer Relationship Management | CRM | Not in SAP — MDM lead |
| Environment, Health & Safety | EH&S | Not in SAP — MDM lead |
| Finance / Controlling | FI/CO | Centrally managed in SAP |
| Human Resources | HR/HCM | Not in SAP — MDM lead |
| Supply Chain / Purchasing | MM/SCM | Centrally managed + some local |
| Sales & Distribution | SD | Split — central and non-SAP |
| Plant Maintenance | PM | Split — central and local |
| Production Planning / Quality | PP/QM | Locally managed in SAP |
| Project Systems | PS | Centrally managed + some local |
| Cross-module / Other | Various | Mixed |

---

## Adding a New Page

### To an existing section

1. Create a new `.md` file in the relevant folder:

    ```
    docs/architecture/my-new-topic.md
    ```

2. Add it to the `nav` in `mkdocs.yml`:

    ```yaml
    - Architecture:
        - MDM Architecture:
            - Lead System Strategy: architecture/lead-system-strategy.md
            - Integration Patterns: architecture/integration-patterns.md
            - RBAC and Governance: architecture/rbac-governance.md
            - Data Enrichment Governance: architecture/data-enrichment-governance.md
            - My New Topic: architecture/my-new-topic.md   # ← add here
    ```

3. Run `mkdocs serve` to preview.

---

### As a new section

1. Create a new folder under `docs/`:

    ```
    docs/decisions/
    ```

2. Add at least one `.md` file inside it:

    ```
    docs/decisions/adr-001-mdm-platform-selection.md
    ```

3. Add the section to `mkdocs.yml`:

    ```yaml
    nav:
      - Home: index.md
      - Goals & Objectives: goals-objectives.md
      - Benefits: benefits.md
      - Assumptions: assumptions.md
      - Scope: ...
      - Requirements: ...
      - Architecture: ...
      - Operations: ...
      - AI in MDM: ai/overview.md
      - Code Reference: ...
      - Decision Records:                                           # ← new section
          - MDM Platform Selection: decisions/adr-001-mdm-platform-selection.md
      - About:
          - Project Structure: project-structure.md
    ```

4. Run `mkdocs serve` to preview.

---

## File Naming Conventions

| Convention | Example | Reason |
|---|---|---|
| Lowercase, hyphen-separated | `lead-system-strategy.md` | URL-friendly, consistent across OS |
| Descriptive, not abbreviated | `integration-patterns.md` not `int-pat.md` | Self-documenting |
| No spaces or underscores | `rbac-governance.md` | Avoids URL encoding issues |
| Prefix ADRs with `adr-NNN-` | `adr-001-platform-selection.md` | Keeps decision records ordered |

---

## Launching the Site

```bash
cd /Users/sbaker/OneDrive/workspace/SierraMDM/Documentation
mkdocs serve
```

Open [http://127.0.0.1:8000](http://127.0.0.1:8000) in your browser. The site live-reloads on every file save.
