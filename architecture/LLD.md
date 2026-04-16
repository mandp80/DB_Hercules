~~~mermaid

sequenceDiagram
    autonumber
    actor User
    participant UI as React SPA
    participant Ctrl as Report Controller
    participant Service as Reporting Service (Spring Batch)
    participant Repo as Repository / DAO
    participant DB as Database
    participant FileSvc as File Management Service

    User->>UI: Click "Download Report"
    UI->>Ctrl: GET /api/v1/reports/download/{id}
    Note right of Ctrl: Authenticated via API Gateway
    
    Ctrl->>Service: initiateReportDownload(reportId)
    Service->>Repo: findReportMetadata(reportId)
    Repo->>DB: SELECT report_metadata
    DB-->>Repo: Metadata (File Name, Type)
    
    Service->>Repo: fetchReportData()
    Repo->>DB: Execute Query (Trade/Cash Data)
    DB-->>Repo: Data Result Set
    
    Note over Service: Processing Data (Spring Batch Tasklet)
    Service->>Service: Generate Excel File (Apache POI)
    
    Service->>FileSvc: storeGeneratedFile(excelFile)
    FileSvc-->>Service: fileStoragePath / UUID
    
    Service-->>Ctrl: InputStreamResource (Excel File)
    Ctrl-->>UI: 200 OK (application/vnd.ms-excel)
    
    UI->>User: Browser Download Prompt (report.xlsx)
