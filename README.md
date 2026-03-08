# Hercules

### Introduction
Deutsche Bank has many financial assets and liabilities like stocks, bonds and loans. Regulators want to know the bank values these items, especially those that are not easily priced in the market (called "Level 3" valuation).

The **Hercules** system is designed and developed to gather all the necessary financial information, proceess it and create 3 specific IFRS 13 reports (Forms C337, C376, C377)

- **IFRS 13:** Mandates the disclosure of financial instrument classification at fair value into a three level hierarchy (Level 1, Level 2 and Level 3).
- **Form C337:** Discloses the Fair Value of Deutsche Bank's Financial Instruments valued using a Level 3 valuation (significant unobservable inputs).
- **Form C376:** Requires a Roll Forward Table for Level 3 Financial Assets and Liabilities, explaining movements ue to fair value changes, cash movements, and transfers between levels.
- **Form C377:** Discloses unrealized P&L on Level 3 financial instruments held at the reporting date.

### Purpose and Scope
The current legacy system is standalone reporting application, which relies on manual processes, IT itervention, email-based data sharing and Excel driven workflow, resulting into security, scalability and operational efficency challenges. To address these gaps, a modern web=based Java/Spring application deployed on Openshift cluster, has been introduced to automate and streamline the entire reporting lifecycle.

### Challenges and Limitation of existing System
- **Unsecure:** No standard protocol is implemented for user login and password is stored in local database in unsecured manner.
- **Operational and Data Risk:** Manual data handling, email based data sharing and direct database updates by IT team.
- **Lack of Audit Control:** No clear end-to-end audit trail, making it difficult to ensure compliance and trace the data lineage.
- **Slow Execution:** Single threaded application resulting into bottleneck when multiple users tries to upload the data and generate the reports.
- **Technology Obsolescence:** As a legacy platform, it is difficult to enhance, scale or migrate, posing long term technology risk.

### Solution
Hercules is introduced to empower users with powerful new features:
- **Automated Scheduler:** Many Data ingestion/loading task is scheduled as well can be run on-demand basis from the UI, removing the dependency from IT team.
- **Monitoring Dashboard:** A real-time dashboard provides complete visibility into the status of all the activities, allowing user to track the progress independently.
- **Automated Notification:** Users will receive automatic email notification upon completion or failure of the task with status.
- **Audit Trail:** All activities perfomed through UI or System are captured in audit trail, ensuring full traceability from source to target.

### Flow Diagram

Below is the flow diagram depicting typical actvities performed by Users to submit IFRS reporting to the Finances.

```mermaid
%%{
  init: {
    'theme': 'base',
    'flowchart': { 'curve': 'stepBefore' },
    'themeVariables': {
      'primaryColor': '#ffffff',
      'primaryBorderColor': '#000000',
      'lineColor': '#000000',
      'fontFamily': 'arial',
      'fontSize': '14px'
    },
    'style': {
      'global': [
        '.node rect, .node circle, .node polygon, .node path { stroke-width: 3px !important; }',
        '.edgePath .path { stroke-width: 3px !important; }'
      ]
    }
  }
}%%
graph TD
    subgraph PreQE ["Pre-Quarter End Activities"]
        UBR("Review & Update UBR Mapping Data") --> SECD(Review & Update Trade Book Mapping)
        SECD --> RollForward("Roll Forward UBR Static")
    end

    subgraph Data["Data Ingestion"]
        RollForward --> Upload{"Upload Data"}
        Upload --> FX("Fx Rates")
        Upload --> Cash("Cash")
        Upload --> Bond("Bond ISIN")
        Upload --> MARUP("UBR Hierarchies")
        Upload --> GVG("Levelling")
        Upload --> Trade("Trade")
        Upload --> Balance("Opening Balance")
        FX --> Uploaded{"Upload Successful"}
        Cash --> Uploaded
        Bond --> Uploaded 
        MARUP --> Uploaded 
        GVG --> Uploaded 
        Trade --> Uploaded 
        Balance --> Uploaded 
    end

    subgraph Processing["System Processing & Output Checks"]
        Uploaded --> ReportGeneration("RFT Report Generation")
        ReportGeneration --> PerformChecks("Perform Opening Balance Check & Reconcile C376")
        PerformChecks --> SignOff("Sign Off Report & Distribute  to Regional Finance")
        SignOff --> PostSubmission("Post-Submission Activities")
    end 

    

    %% High-contrast styling
    classDef pink fill:#FFD3DA,stroke:#333,stroke-dasharray: 2 2,stroke-width:3px
    classDef skyblue fill:#D6F7FF,stroke:#333,stroke-dasharray: 2 2,stroke-width:3px
    classDef yellow fill:#fff9c4,stroke:#333,stroke-dasharray: 2 2,stroke-width:3px
    classDef node stroke:#333,stroke-width:2px
   
    linkStyle default stroke:#333,stroke-width:2px;

    class PreQE pink;
    class Data skyblue;
    class Processing yellow;
