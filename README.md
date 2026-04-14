graph TB
    subgraph CLIENT["👤 Client Layer"]
        U["User\n(Browser)"]
    end

    subgraph UI["🖥️ Reporting UI Layer\n(React — GKE)"]
        LOGIN["Authentication\n& SSO Login"]
        CATALOGUE["Report Catalogue\n(Browse & Select)"]
        FILTERS["Filter Panel\n(Date Range, Account\nStatus, Segment etc.)"]
        VALIDATOR["UI Input Validator\n(Mandatory Fields\n& Format Checks)"]
        VIEWER["Report Viewer\n(Download PDF / CSV)"]
    end

    subgraph ORCH["⚙️ Report Orchestration Layer\n(Spring Boot — GKE)"]
        ORCH_CTRL["Report Execution\nController\n(Single Entry Point)"]
        ORCH_AUTH["Access & Authority\nValidator\n(Role, Mandate,\nPermitted Accounts)"]
        ORCH_EXEC["Execution Planner\n(Determines how\nreport runs)"]
        ORCH_PAGE["Pagination &\nSorting Handler"]
        ORCH_SCHED["Scheduling\nHandler\n(On-demand /\nScheduled)"]
        ORCH_ADAPT["Renderer\nAdapter\n(Selects PDF\nor CSV renderer)"]
    end

    subgraph SEMANTIC["🧠 Semantic Layer\n(Business Logic Engine — GKE)"]
        SEM_RESOLVE["Report Contract\nResolver\n(Identifies correct\nreport definition)"]
        SEM_LOGIC["Business Logic\nProcessor\n(Domain entities,\nmetrics, calculations)"]
        SEM_FIELDS["Field Definition\nEnforcer\n(Consistent metric\ndefinitions)"]
        SEM_QUERY["Semantic Query\nBuilder\n(Tool-agnostic\ndata requirements)"]
    end

    subgraph DWH["🗄️ Data Warehouse Layer\n(BigQuery — GCP)"]
        BQ_CURATED["Curated Datasets\n(Denormalised,\nPartitioned,\nReport-Optimised)"]
        BQ_LEGACY["Legacy Journey\nData"]
        BQ_GCP["GCP Modernised\nJourney Data"]
        BQ_PERMS["Permissions &\nAudit Data"]
    end

    subgraph RENDERER["🖨️ Renderer Layer\n(Report Rendering Engine — GKE)"]
        TMPL["Layout Template\nEngine\n(JRXML / HTML\nTemplates)"]
        PDF_R["PDF Renderer\n(JasperReports /\niText)"]
        CSV_R["CSV Renderer\n(Apache POI /\nOpenCSV)"]
        GCS["Cloud Storage\n(GCS Bucket)\nSigned URL Output"]
    end

    subgraph SECURITY["🔐 Security & Gateway"]
        APIGEE["Apigee API Gateway\n(OAuth2 / JWT\nRate Limiting\nAudit Logging)"]
        IAP["Cloud IAP\n(Identity Aware\nProxy — SSO)"]
        VAULT["HashiCorp Vault\n(Secrets)"]
    end

    %% User Flow
    U --> IAP --> LOGIN
    LOGIN --> CATALOGUE
    CATALOGUE --> FILTERS
    FILTERS --> VALIDATOR
    VALIDATOR -->|"Report Request\n(reportId + filters\n+ security context)"| APIGEE

    %% Orchestration
    APIGEE --> ORCH_CTRL
    ORCH_CTRL --> ORCH_AUTH
    ORCH_AUTH --> ORCH_EXEC
    ORCH_EXEC -->|"Report ID + Authorised\nFilters + User Context"| SEM_RESOLVE

    %% Semantic Layer
    SEM_RESOLVE --> SEM_LOGIC
    SEM_LOGIC --> SEM_FIELDS
    SEM_FIELDS --> SEM_QUERY
    SEM_QUERY -->|"Optimised\nBigQuery Query"| BQ_CURATED

    %% Data Warehouse
    BQ_LEGACY --> BQ_CURATED
    BQ_GCP --> BQ_CURATED
    BQ_PERMS --> BQ_CURATED
    BQ_CURATED -->|"Filtered Partition-aware\nReport-Ready Dataset"| SEM_FIELDS

    %% Semantic Response back
    SEM_FIELDS -->|"Semantic Response\n(Business-correct data)"| ORCH_EXEC

    %% Orchestration Rendering prep
    ORCH_EXEC --> ORCH_PAGE
    ORCH_PAGE --> ORCH_SCHED
    ORCH_SCHED --> ORCH_ADAPT
    ORCH_ADAPT -->|"Paginated +\nSorted Data\n+ Template Config"| TMPL

    %% Renderer
    TMPL --> PDF_R
    TMPL --> CSV_R
    PDF_R --> GCS
    CSV_R --> GCS
    GCS -->|"Signed URL"| ORCH_CTRL

    %% Return to UI
    ORCH_CTRL -->|"Report Artifact\n(Signed URL)"| APIGEE
    APIGEE --> VIEWER
    VIEWER --> U

    %% Security
    VAULT -.->|"Secrets"| ORCH_CTRL
    VAULT -.->|"Secrets"| SEM_QUERY

    %% Styles
    classDef clientStyle fill:#E8F5E9,stroke:#2E7D32,color:#1B5E20,font-weight:bold
    classDef uiStyle fill:#E3F2FD,stroke:#1565C0,color:#0D47A1,font-weight:bold
    classDef orchStyle fill:#FFF3E0,stroke:#E65100,color:#BF360C,font-weight:bold
    classDef semStyle fill:#F3E5F5,stroke:#6A1B9A,color:#4A148C,font-weight:bold
    classDef dwhStyle fill:#E0F7FA,stroke:#00695C,color:#004D40,font-weight:bold
    classDef rendStyle fill:#FCE4EC,stroke:#880E4F,color:#560027,font-weight:bold
    classDef secStyle fill:#FFF8E1,stroke:#F57F17,color:#E65100,font-weight:bold

    class U clientStyle
    class LOGIN,CATALOGUE,FILTERS,VALIDATOR,VIEWER uiStyle
    class ORCH_CTRL,ORCH_AUTH,ORCH_EXEC,ORCH_PAGE,ORCH_SCHED,ORCH_ADAPT orchStyle
    class SEM_RESOLVE,SEM_LOGIC,SEM_FIELDS,SEM_QUERY semStyle
    class BQ_CURATED,BQ_LEGACY,BQ_GCP,BQ_PERMS dwhStyle
    class TMPL,PDF_R,CSV_R,GCS rendStyle
    class APIGEE,IAP,VAULT secStyle
    
