# 📐 Architectural Atlas
## Platform Partners — ServiceTitan Data Platform

> **Purpose:** This document provides a complete visual map of the Data Platform. It is structured from a high-level executive overview down to technical implementation detail. It serves as the single source of truth for understanding, maintaining, and replicating the platform.

---

## 1. 🗺️ High-Level System Overview

This is the "30,000 ft view" — the entire platform in one diagram.

```mermaid
graph TD
    subgraph SOURCES["📡 DATA SOURCES"]
        ST["🔧 ServiceTitan<br/>(Field Operations SaaS)"]
        FV["🔗 Fivetran<br/>(Managed Connector)"]
    end

    subgraph INGESTION["⚙️ INGESTION LAYER"]
        ETL["🚀 Custom ETL<br/>Cloud Run Jobs"]
        FV_BQ["📥 Fivetran → BigQuery<br/>(Direct Sync)"]
    end

    subgraph STORAGE["🏗️ DATA LAKE — BigQuery (per Tenant)"]
        BZ["🥉 Bronze Dataset<br/>(Raw Data)"]
        SV["🥈 Silver Dataset<br/>(Transformed Views)"]
        DB["🖥️ Dashboards Dataset<br/>(Business Views)"]
    end

    subgraph ANALYTICS["📊 ANALYTICS & CONSUMPTION"]
        DF["⚙️ Dataform<br/>(SQL Transformations)"]
        GS["📋 Google Sheets<br/>(Operational Reports)"]
        LS["📈 Looker Studio<br/>(Executive Dashboards)"]
        PY["🧠 Python / Streamlit<br/>(Predictive Analytics)"]
    end

    subgraph MGMT["🛠️ PLATFORM MANAGEMENT (Central)"]
        GCA["☁️ GCloud Automation<br/>(IaC Scripts)"]
        MON["🔍 Monitoring<br/>(ETL Health Dashboard)"]
        CFG["⚙️ Settings & Config<br/>(Multi-Tenant Registry)"]
    end

    ST --> ETL
    ST --> FV --> FV_BQ
    ETL --> BZ
    FV_BQ --> BZ
    BZ --> SV
    SV --> DF --> DB
    DB --> GS
    DB --> LS
    BZ --> PY
    SV --> PY

    GCA --> STORAGE
    CFG --> ETL
    MON --> ETL
```

---

## 2. ⚙️ ETL Deep-Dive: Data Extraction Pipeline

This diagram shows the exact internal orchestration of the custom ETL process from source API to BigQuery.

```mermaid
sequenceDiagram
    participant SCH as ☁️ Cloud Scheduler
    participant ORC as 🎯 Cloud Function<br/>(Orchestrator)
    participant J1 as 🚀 Cloud Run Job #1<br/>(st2json)
    participant J2 as 🔄 Cloud Run Job #2<br/>(json2bq)
    participant ST as 🔧 ServiceTitan API
    participant GCS as 🪣 Cloud Storage<br/>(GCS Bucket)
    participant BQ as 🗄️ BigQuery<br/>(Bronze Dataset)

    SCH->>ORC: Trigger (every 6h)
    ORC->>J1: Execute Job #1
    loop For each Tenant (Company)
        J1->>ST: GET /api/endpoint?page=N
        ST-->>J1: JSON Response (paginated)
        J1->>GCS: Upload JSON file
    end
    J1-->>ORC: ✅ Completed
    ORC->>J2: Execute Job #2
    loop For each JSON file in GCS
        J2->>GCS: Download JSON
        J2->>J2: Transform → JSONL + snake_case
        J2->>BQ: MERGE into Bronze Table
    end
    J2-->>ORC: ✅ Completed
    ORC-->>SCH: 200 OK
```

**Key behaviors:**
- 🔁 **Incremental MERGE:** Only inserts/updates changed records, never full reloads
- 🏷️ **Audit Fields:** Every record gets `_etl_synced` (timestamp) and `_etl_operation` (INSERT/UPDATE/DELETE)
- 🧹 **Soft Delete:** Deleted records are flagged, never physically removed

---

## 3. 🏢 Multi-Tenant Architecture

Each client company is a fully isolated tenant with its own GCP project and BigQuery datasets.

```mermaid
graph LR
    subgraph CENTRAL["🧠 Central Management Project<br/>(pph-central)"]
        META["📋 metadata_consolidated_tables<br/>(Endpoint Registry)"]
        LOGS["📜 ETL Logs<br/>(Cross-Tenant Monitoring)"]
        SA["🔑 Service Accounts<br/>(etl-servicetitan@...)"]
    end

    subgraph T1["🏠 Tenant A<br/>(shape-mhs-1)"]
        BZ1["Bronze Dataset"]
        SV1["Silver Dataset"]
        DB1["Dashboards Dataset"]
    end

    subgraph T2["🏠 Tenant B<br/>(pph-inbox)"]
        BZ2["Bronze Dataset"]
        SV2["Silver Dataset"]
        DB2["Dashboards Dataset"]
    end

    subgraph TN["🏠 Tenant N<br/>(...)"]
        BZN["Bronze Dataset"]
        SVN["Silver Dataset"]
        DBN["Dashboards Dataset"]
    end

    META --> T1
    META --> T2
    META --> TN
    SA --> T1
    SA --> T2
    SA --> TN
    LOGS -.->|monitors| T1
    LOGS -.->|monitors| T2
    LOGS -.->|monitors| TN
```

> [!IMPORTANT]
> The `pph-central` project is the control tower. All automation, IAM policies, and metadata configurations are managed from here and pushed out to each tenant project.

---

## 4. 🌎 Multi-Environment (SDLC) Structure

Changes flow through 3 environments before reaching clients, ensuring stability.

```mermaid
graph LR
    DEV["🔧 DES<br/>Development<br/>platform-partners-des<br/><br/>Free to break things"]
    QUA["🧪 QUA<br/>Quality / Staging<br/>platform-partners-qua<br/><br/>Validation before prod"]
    PRO["🚀 PRO<br/>Production<br/>constant-height-455614-i0<br/><br/>Live clients"]

    DEV -- "build_deploy.sh des" --> QUA
    QUA -- "build_deploy.sh pro" --> PRO

    style DEV fill:#4a9eff,color:#fff
    style QUA fill:#f0a500,color:#fff
    style PRO fill:#2ecc71,color:#fff
```

**Deployment automation:** Each environment has its own Cloud Run Jobs, schedulers, and service accounts. Promotion between environments is a single shell command (`build_deploy.sh [env]`).

---

## 5. 🗄️ Data Layer Architecture (Medallion)

Data quality and readiness increases as it moves through each layer.

```mermaid
graph TD
    subgraph BZ["🥉 BRONZE — Raw Ingestion"]
        BR1["📄 Raw tables from ServiceTitan API<br/>e.g. timesheet, technician, campaign..."]
        BR2["🏷️ + audit fields: _etl_synced, _etl_operation"]
    end

    subgraph SV["🥈 SILVER — Business Transformations"]
        SV1["🔗 Joined & enriched views<br/>e.g. vw_dailytracker_timestamp_base"]
        SV2["📐 Business logic applied<br/>(timezone conversion, overtime rules...)"]
    end

    subgraph DB["🖥️ DASHBOARDS — Consumption Ready"]
        DB1["📊 LTM Views (Last 12 Months KPIs)"]
        DB2["📈 PULSE Views (Operational Metrics)"]
        DB3["📅 Daily Tracker Views (Payroll & Hours)"]
    end

    subgraph FV["🔗 FIVETRAN — Parallel Raw Layer"]
        FV1["Tables synced directly<br/>e.g. estimates, invoices, jobs..."]
    end

    BZ --> SV --> DB
    FV --> SV
    FV --> DB
```

---

## 6. 🛠️ Platform Management: Automation Capabilities

The platform includes a full suite of automation scripts (Infrastructure as Code) for deploying to new tenants.

```mermaid
mindmap
  root((☁️ GCloud Automation))
    BigQuery
      Create Datasets
        bronze
        silver
        dashboards
      Set IAM Permissions
      Create Authorized Views
    IAM & Security
      Service Account Setup
      Custom Roles
        pphSheetsAnalyst
        pphBronzeReader
      Permission Templates
    ETL Deployment
      Build Docker Images
      Deploy Cloud Run Jobs
      Configure Schedulers
    Monitoring
      ETL Health Dashboard
      Table Count Alerts
      Cross-Tenant Log Review
    Settings
      Tenant Registry
      Company Timezone Config
      Business Unit Mapping
```

---

## 7. 📊 Analytics & Products Built on the Platform

What do clients actually get? These are the visible outputs.

```mermaid
graph LR
    subgraph REPORTS["📊 Operational Reports (Google Sheets)"]
        LTM["📈 LTM Dashboard<br/>Last 12 Months KPIs<br/>Revenue, Jobs, Conversion"]
        PULSE["⚡ PULSE Dashboard<br/>Real-time Business Metrics"]
        DT["📅 Daily Tracker<br/>Employee Hours & Payroll"]
    end

    subgraph ANALYTICS["🧠 Advanced Analytics (Python/Streamlit)"]
        CALL["📞 Call Temperature Analysis<br/>Lead quality scoring"]
        PRED["🔮 Predictive Models<br/>Historical variability<br/>Demand forecasting"]
    end

    subgraph LOOKER["📈 Executive Dashboards (Looker Studio)"]
        EXEC["📋 Multi-Company Overview"]
    end

    DB["🖥️ Dashboards Dataset<br/>(BigQuery)"] --> LTM
    DB --> PULSE
    DB --> DT
    DB --> EXEC
    BZ["🥉 Bronze Dataset<br/>(BigQuery)"] --> CALL
    BZ --> PRED
```

---

## 8. 🔄 Platform Component Dependency Map

*For maintenance: know what breaks if one component fails.*

```mermaid
graph TD
    ST_API["🔧 ServiceTitan API"] -->|"fails → no fresh data"| J1["Cloud Run Job #1<br/>(st2json)"]
    J1 -->|"fails → json2bq has nothing"| GCS["GCS Bucket<br/>(JSON files)"]
    GCS -->|"files missing → tables not updated"| J2["Cloud Run Job #2<br/>(json2bq)"]
    J2 -->|"fails → Bronze stale"| BZ["Bronze Tables<br/>(BigQuery)"]
    BZ -->|"stale → Silver stale"| SV["Silver Views<br/>(Dataform)"]
    SV -->|"stale → Dashboards stale"| DB["Dashboard Views"]
    DB -->|"stale → client reports wrong"| GS["Google Sheets<br/>/ Looker"]

    META["📋 metadata_consolidated_tables"] -->|"missing endpoints → J1 uses fallback"| J1
    ORC["Cloud Function<br/>(Orchestrator)"] -->|"timeout → J2 might not trigger"| J2

    style ST_API fill:#e74c3c,color:#fff
    style GS fill:#2ecc71,color:#fff
```

> [!WARNING]
> The highest-risk single point of failure is **Cloud Function timeout**. If the orchestrator times out between Job 1 and Job 2, the pipeline is only half-executed with no automatic recovery. This is addressed in the Technical Debt plan.

---

*Last updated: March 10, 2026 — Platform Partners Data Team*
