## Team Handoffs Summary

Product coordinates requirements with all teams from the center. Two paths exist: External and Internal datasets.

```mermaid
graph TD
    PRODUCT[PRODUCT TEAM<br/>Central Coordinator]

    subgraph EXTERNAL[" EXTERNAL DATASET PATH "]
        A1_EXT[A1 TEAM<br/>Ingesting, Cleaning,<br/>Enriching, Applying<br/>Business Graph Logic]
    end

    subgraph INTERNAL[" INTERNAL DATASET PATH "]
        INT_TEAM[Internal Bloomberg Team]
    end

    AM[A&M TEAM<br/>Data Transformation & Service Config]
    TI[TI TEAM<br/>BQL, Entitlements & Ingestion]
    UI[UI TEAM<br/>Service Integration & Display]

    READY[All Services Ready]
    USERS[ALTD Clients]

    PRODUCT -->|Data Requirements| A1_EXT
    PRODUCT -->|Data Requirements| INT_TEAM
    PRODUCT -->|Metadata Requirements<br/>Mapping & Methodology Coordination| AM
    PRODUCT -->|BQL & Entitlement Requirements| TI
    PRODUCT -->|UI Requirements| UI

    A1_EXT -->|S3 files| AM

    AM -->|Business graph logic| INT_TEAM
    INT_TEAM -->|Aggregated/raw data| TI

    AM -->|Transformed files<br/>Configure metrics<br/>Data Delivery| TI

    TI -->|Make data accessible| READY
    READY --> UI
    UI --> USERS

    style PRODUCT fill:#fff9c4,stroke:#f57f17,stroke-width:3px
    style AM fill:#4fc3f7,stroke:#0277bd,stroke-width:3px
    style TI fill:#ce93d8,stroke:#6a1b9a,stroke-width:3px
    style UI fill:#81c784,stroke:#2e7d32,stroke-width:3px
    style EXTERNAL fill:#f5f5f5,stroke:#666,stroke-width:2px,color:#000
    style INTERNAL fill:#f5f5f5,stroke:#666,stroke-width:2px,color:#000
```

## A&M Team: Detailed Onboarding Workflow (Stage-Based)

This diagram shows A&M's work organized by stages with repos, services, and tables that get updated.

```mermaid
graph TD
    subgraph INPUT[" INPUT FROM A1 / INTERNAL TEAM "]
        A1_FILES["AWS S3 Files from A1<br/>business-graph-metadata-v2<br/>&lt;dataset&gt; raw-clean data"]
        INT_DATA["Internal Dataset<br/>From Internal Bloomberg Team<br/> "]
    end

    subgraph STAGE1[" "]
        direction TB
        STAGE1_TITLE["STAGE 1: DATA DELIVERY - External Only"]

        subgraph OFF_PREM[" OFF-PREMISES "]
            S1_DECISION{Decide Calculation Type}

            S1_DP_OTF[OLAP Data-plane]
            S1_DEPLOY_OTF[Deployment]
            S1_PUBSET_OTF[Pubset]

            S1_DP_PRE[Aggregator Data-plane]
            S1_DEPLOY_PRE[Deployment]
            S1_PUBSET_PRE[Pubset]
        end

        subgraph ON_PREM[" ON-PREMISES "]
            S1_REPO["altd-kafka-consumer repo"]
            S1_KAFKA["Kafka Topic"]
            S1_CH_HANDLER["Handler: ClickHouse Ingestion<br/>→ Argo Workflow → ClickHouse"]
            S1_DS_HANDLER["Handler: Delivery to BCS<br/>→ Argo Workflow → BCS Buckets"]
            S1_KTD_REPO["deepwater-transform repo"]
            S1_KTD_BOX["Key Trend Drivers"]
            S1_KTD_HANDLER["KTD Data Creation/Delivery<br/>→ Argo Workflow → BCS Buckets"]
        end
    end

    subgraph STAGE2[" "]
        direction TB
        STAGE2_TITLE["STAGE 2: SERVICE CONFIGURATION"]
        S2_DATASVC_REPO["altddatasvc repo<br/>Add configuration support<br/>Add logic for observed metrics needing extra support<br/>Update KPI estimates methodology for special metrics"]
        S2_WORKFLOWS_REPO["altd-data-workflows repo<br/>Add mapping info from Product<br/>Add metadata (metrics, breakouts, periodicities)"]
        S2_DATASVC_SVC["altddatasvc service"]
    end

    subgraph STAGE3[" "]
        direction TB
        STAGE3_TITLE["STAGE 3: TABLE UPDATES & WORKFLOWS - Strict Order"]

        S3_DATASET["altd_datasets<br/>PRQS SQL manual"]

        S3_META_WF["Metadata Workflow<br/>PRQS EX manual"]
        S3_META_TABLES["altd_metrics_metadata<br/>altd_metrics_breakouts<br/>altd_metrics_periodicities"]

        S3_MAP_WF["Mapping Workflow<br/>Scheduled 3pm daily"]
        S3_MAP_TABLE["cofi_altd_mapping"]

        S3_KPI_WF["KPI Recommendation<br/>Scheduled 5pm daily"]
        S3_KPI_TABLE["kpi_recommendation_scores"]
    end

    subgraph TI_INFRA[" TI INFRASTRUCTURE "]
        TI_INGEST["TI Team:<br/>Ingest to COMDB2 & BHATS"]
        TI_BQL["TI Team:<br/>Support data in BQL"]
    end

    subgraph STAGE4[" "]
        STAGE4_TITLE["STAGE 4: AUTO-PROPAGATION - TI: metadatasvc & datasvc | UI: altdsvc"]
        S4_METASVC["altdmetadatasvc"]
        S4_DATASVC["altddatasvc"]
        S4_ALTSVC["altdsvc"]
    end

    subgraph STAGE5[" STAGE 5: UI READINESS "]
        UI_READY["UI Team:<br/>Calls altdsvc to pull data<br/>Display on ALTD GO"]
    end

    A1_FILES --> S1_DECISION
    INT_DATA --> S2_DATASVC_REPO

    S1_DECISION -->|On-the-fly| S1_DP_OTF
    S1_DECISION -->|Pre-computed| S1_DP_PRE

    S1_DP_OTF --> S1_DEPLOY_OTF
    S1_DEPLOY_OTF --> S1_PUBSET_OTF
    S1_PUBSET_OTF --> S1_REPO

    S1_DP_PRE --> S1_DEPLOY_PRE
    S1_DEPLOY_PRE --> S1_PUBSET_PRE
    S1_PUBSET_PRE --> S1_REPO

    S1_REPO --> S1_KAFKA
    S1_KAFKA --> S1_CH_HANDLER
    S1_KAFKA --> S1_DS_HANDLER

    S1_CH_HANDLER --> S2_DATASVC_REPO
    S1_DS_HANDLER --> TI_INGEST
    S1_DS_HANDLER --> S2_DATASVC_REPO

    S1_KTD_REPO --> S1_KTD_BOX
    S1_KTD_BOX --> S1_KTD_HANDLER
    S1_KTD_HANDLER --> TI_INGEST

    TI_INGEST --> TI_BQL

    S2_DATASVC_REPO --> S2_DATASVC_SVC
    S2_WORKFLOWS_REPO --> S3_META_WF

    S3_DATASET --> S3_META_WF
    S3_META_WF --> S3_META_TABLES
    S3_META_TABLES --> S3_MAP_WF
    S3_MAP_WF --> S3_MAP_TABLE
    S3_MAP_TABLE --> S3_KPI_WF
    S3_KPI_WF --> S3_KPI_TABLE

    S3_META_TABLES --> S4_METASVC
    S3_MAP_TABLE --> S4_METASVC
    S3_KPI_TABLE --> S4_METASVC

    S2_DATASVC_SVC --> S4_DATASVC
    TI_BQL --> S4_DATASVC

    S4_METASVC --> S4_ALTSVC
    S4_DATASVC --> S4_ALTSVC

    S4_ALTSVC --> UI_READY

    style STAGE1_TITLE fill:#e3f2fd,stroke:#1976d2,stroke-width:0px,color:#1976d2
    style STAGE2_TITLE fill:#e3f2fd,stroke:#1976d2,stroke-width:0px,color:#1976d2
    style STAGE3_TITLE fill:#fff3e0,stroke:#f57c00,stroke-width:0px,color:#f57c00
    style STAGE4_TITLE fill:#ffebee,stroke:#c62828,stroke-width:0px,color:#c62828
    style STAGE1 fill:#e3f2fd,stroke:#1976d2,stroke-width:2px,color:#000
    style STAGE2 fill:#e3f2fd,stroke:#1976d2,stroke-width:2px,color:#000
    style STAGE3 fill:#fff3e0,stroke:#f57c00,stroke-width:2px,color:#000
    style STAGE4 fill:#ffebee,stroke:#c62828,stroke-width:2px,color:#000
    style STAGE5 fill:#e8f5e9,stroke:#388e3c,stroke-width:2px,color:#000
    style TI_INFRA fill:#e0e0e0,stroke:#757575,stroke-width:2px,color:#000
    style OFF_PREM fill:#f5f5f5,stroke:#666,stroke-width:2px,color:#000
    style ON_PREM fill:#f5f5f5,stroke:#666,stroke-width:2px,color:#000
    style S1_REPO fill:#1e88e5,stroke:#0d47a1,stroke-width:3px,color:#fff
    style S1_KTD_REPO fill:#1e88e5,stroke:#0d47a1,stroke-width:3px,color:#fff
    style S1_KTD_BOX fill:#000,stroke:#000,stroke-width:3px,color:#fff
    style S2_DATASVC_REPO fill:#1e88e5,stroke:#0d47a1,stroke-width:3px,color:#fff
    style S2_WORKFLOWS_REPO fill:#1e88e5,stroke:#0d47a1,stroke-width:3px,color:#fff
    style S4_METASVC fill:#ce93d8,stroke:#6a1b9a,stroke-width:3px
    style S4_DATASVC fill:#ce93d8,stroke:#6a1b9a,stroke-width:3px
    style S4_ALTSVC fill:#ce93d8,stroke:#6a1b9a,stroke-width:3px
```
