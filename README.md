# DB_Hercules

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
        Upload --> Cash("ACS Cash")
        Upload --> Bond("dbTrader Bond ISIN")
        Upload --> MARUP("MARUP UBR Hierarchies")
        Upload --> GVG("GVG Levelling")
        Upload --> Trade("FDW CRES Trade")
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
    classDef pink fill:#FFD3DA,stroke:#333,stroke-dasharray: 2 2
    classDef skyblue fill:#D6F7FF,stroke:#333,stroke-dasharray: 2 2
    classDef yellow fill:#fff9c4,stroke:#333,stroke-dasharray: 2 2

    class PreQE pink;
    class Data skyblue;
    class Processing yellow;
