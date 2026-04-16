# Low Level Design for Report Download Microservice:

## Key Highlights of this Flow:
- **Data Retrieval:** The Service layer interacts with the Repository to fetch the specific Trade or Cash data you've ingested.

- **Excel Generation:** The "Service" block handles the transformation of the database result set into an Excel binary (typically using Apache POI in a Spring environment).

- **File Management:** I included the File Management Service from your HLD, as it's best practice to store a copy of the generated report for audit purposes before streaming it back to the user.

- **Response Type:** The controller returns an InputStreamResource with the appropriate MIME type for Excel to ensure the browser handles the download correctly.

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
~~~
--------

# Low-Level Design (LLD): ACS Cash Ingestion Job

## 1. Overview
The **ACS Cash Job** (Computron Interface) is a sophisticated multi-step batch orchestration designed to migrate and enrich financial data from the **Computron** source database to the **Cerberus (RFT)** reporting layer. The job handles data acquisition, staging, complex business validation, and final archival.

## 2. Technical Stack
* **Framework**: Spring Batch (Java)
* **Architecture**: Mix of Tasklet-oriented and Chunk-oriented processing
* **Data Sources**: Computron (Input), Cerberus/ADS (Staging), RFT (Final Output)
* **Infrastructure**: On-premises OpenShift Cluster

---

## 3. Detailed Workflow (Step-by-Step)

The job is divided into four distinct phases across 11 ordered steps.

### Phase 1: Initialization & Staging
| Step | Type | Logic | Purpose |
| :--- | :--- | :--- | :--- |
| **0** | Tasklet | `getReportingDates()` | Retrieves `cobDate` from DB to use as a global filter for ingestion. |
| **1** | Tasklet | `truncateImpComputron()` | Cleans the `Imp_Computron` staging table to ensure idempotency. |
| **2** | Chunk | `masterComputronToCres()` | **Reader**: Computron DB. **Writer**: `Imp_Computron`. Maps LedgerDTO to ImpComputron. |
| **3** | Tasklet | `spValidateComputron()` | Validates PK integrity across class, indexKey, postDtc, journal, and batchNum. |

### Phase 2: Transformation & Enrichment
| Step | Type | Logic | Purpose |
| :--- | :--- | :--- | :--- |
| **4** | Tasklet | `updatesPreSpInsertCash()` | Truncates ADS staging tables and sanitizes `tref3` column data. |
| **5** | Chunk | `spInsertCash()` | **The Enrichment Engine**: Migrates data from `Imp_Computron` to `ads.dbo.Cash`. |

> **Note on Step 5 Enrichment**: The ItemProcessor performs complex lookups across `Trade_Book`, `Calendar`, `Currency`, and `Rate` tables to calculate `eur_total` and validate `book_id`.

### Phase 3: Business Logic & Bond Processing
| Step | Type | Logic | Purpose |
| :--- | :--- | :--- | :--- |
| **6** | Tasklet | `spInsertBonds()` | Aggregates Buy/Sell records where net amount is between -1 and 1; inserts to `ads.dbo.Bonds`. |
| **7** | Tasklet | `updateFinalCashData()` | Joins Cash and Bonds data to apply `buy_sell_adjustment` flags. |
| **8** | Tasklet | `spReviewFX()` | **Quality Gate**: Validates FX Rates. Throws `Existing Data Exception` if rates are missing. |

### Phase 4: Final Migration & Archival
| Step | Type | Logic | Purpose |
| :--- | :--- | :--- | :--- |
| **9** | Tasklet | `preSpAppendCash()` | **Safety Check**: Ensures no existing records exist for the `cobDate` in the final table. |
| **10**| Chunk | `spAppendCash()` | Final migration from `ads.dbo.Cash` (ADS) to `rft.dbo.Cash` (Reporting). |

---

## 4. Sequence Diagram

~~~mermaid
sequenceDiagram
    autonumber
    participant Job as ComputronInterfaceJob
    participant DB_In as Computron (Source)
    participant Imp as Imp_Computron (Staging)
    participant ADS as ADS DB (Processing)
    participant RFT as RFT DB (Final)

    rect rgb(240, 240, 240)
    Note over Job, DB_In: Phase 1: Setup & Ingest
    Job->>DB_In: Step 0: Get cobDate
    Job->>Imp: Step 1: Truncate Staging
    Job->>DB_In: Step 2: Extract & Map Data
    Job->>Imp: Write to Imp_Computron
    Job->>Imp: Step 3: Validate Primary Keys
    end

    rect rgb(230, 245, 255)
    Note over Job, ADS: Phase 2: Enrichment
    Job->>ADS: Step 4: Prepare ADS Tables
    Job->>Imp: Step 5: Read Staging Data
    Note right of Job: Lookup: Book, Calendar, FX Rates
    Job->>ADS: Write enriched data to ads.dbo.Cash
    end

    rect rgb(240, 240, 240)
    Note over Job, ADS: Phase 3: Logic & Validation
    Job->>ADS: Step 6: Process Bonds (Buy/Sell Joins)
    Job->>ADS: Step 7: Apply Adjustments
    Job->>ADS: Step 8: FX Quality Review (Fail-Fast)
    end

    rect rgb(255, 245, 230)
    Note over Job, RFT: Phase 4: Final Append
    Job->>RFT: Step 9: Duplicate Date Check
    Job->>ADS: Step 10: Read Enriched ADS Data
    Job->>RFT: Write to rft.dbo.Cash (Final)
    end
~~~
---------------

# Low-Level Design (LLD): Trade Data Ingestion (FDW Oracle to RFT)

## 1. Overview
The **Trade Data Ingestion Job** is a high-performance Spring Batch implementation designed to migrate massive datasets from the **FDW Oracle (Source)** database to the **Cerberus (RFT)** reporting ecosystem. It is architected to handle enterprise-scale data (100,000+ records) using parallel processing and a multi-layered transformation pipeline.

## 2. Technical Execution Strategy
To minimize the ingestion window, the job utilizes a **Master-Slave Partitioning** strategy:

* **Grid Size (Parallelism)**: Defined in the Kubernetes `ConfigMap`, currently set to **10 threads**. The dataset is divided into 10 partitions of ~10,000 records each, processed simultaneously.
* **Chunk Size (Transactions)**: Set to **500 records**. The system performs a database commit after every chunk, ensuring a balance between high throughput and transactional safety.
* **Performance Metrics**: 
    * Total Records: 100,000
    * Total Chunks: 200 per Partition
    * Commit Interval: Every 500 records

---

## 3. Workflow Sequence Diagram

~~~mermaid
sequenceDiagram
    autonumber
    participant Job as Trade Ingestion Job
    participant FDW as FDW Oracle (Source)
    participant ADS as ADS DB (Staging)
    participant RFT as RFT DB (Reporting)
    participant Cache as In-Memory Cache

    rect rgb(240, 240, 240)
    Note over Job, ADS: Phase 1: Ingestion & Staging
    Job->>ADS: Step 1: Truncate Ingestion & Trade Tables
    Job->>FDW: Step 2 & 3: Partitioned Read (SRC_TRN / SRC_FAC)
    FDW-->>Job: Data Streams (10 Parallel Threads)
    Job->>ADS: Write to imp_Cres / imp_Cres_Fac (Chunk: 500)
    Job->>ADS: Step 4: Cross-table Trade Validation
    end

    rect rgb(230, 245, 255)
    Note over Job, Cache: Phase 2: High-Performance Flow (Step 5)
    Job->>ADS: Drop Indexes (Maximize Write Speed)
    Job->>Cache: Load Reference Tables (Lookups)
    Job->>ADS: Migrate to ads.dbo.Trade & Re-create Indexes
    Job->>ADS: Update TradeIFRS & Manual Adjustments
    Job->>Cache: Clear Memory Cache
    end

    rect rgb(240, 240, 240)
    Note over Job, RFT: Phase 3: Reporting & Financial Engines
    Job->>RFT: Step 6: Reporting Date Safety Gate
    Job->>ADS: Step 7 & 8: Migrate ADS to RFT Staging
    Job->>RFT: Step 9 & 10: Run Netting & GVG Engines
    Job->>RFT: Step 11: Set Global flowLink Context
    end

    rect rgb(255, 245, 230)
    Note over Job, RFT: Phase 4: Final Aggregation (RFT Engine)
    Job->>RFT: Step 12: Complex RFT Aggregation Flow
    Note right of Job: Parallel Processing (Q1, Q2, Q3)
    Job->>RFT: Consolidate into RFT_Table_Summ
    Job->>RFT: Final Write to RFT_Y_Summ / RFT_Q_Summ
    end
~~~

## 4. Step-by-Step Technical Description

**Phase 1: Preparation & Staging**

- **Step 1:** truncateImpCresImpCresFacAndTrade Tasklet to ensure a clean state by truncating imp_Cres, imp_Cres_Fac, and the Trade processing tables.

- **Step 2:** masterInputToCres Migrates data from SRC_TRN_DET (Oracle) to imp_Cres. Uses partitioned readers for parallel extraction.

- **Step 3:** masterInputToCresFac Migrates data from SRC_FAC_DET to imp_Cres_Fac.

- **Step 4:** validateTrade Validates data integrity and performs initial updates between landing and staging layers.

**Phase 2: Optimization Flow (masterCresToTradeStepWithIndex)**

- **Step 5:** Performance Flow A complex internal flow consisting of:
    
  - **Index Management:** Drops indexes before heavy writes and recreates them afterward to optimize I/O.
    
  - **In-Memory Lookups:** Loads small reference tables into memory to eliminate database round-trips for metadata lookups.
    
  - **Data Promotion:** Migrates data from ads.dbo.Imp_Cres to ads.dbo.Trade.
    
  - **IFRS Updates:** Updates TradeIFRS and TradeIFRSManual tables.

**Phase 3: Financial Engines & Quality Gates**

  - **Step 6:** checkCobDtInTradeRFT Safety Tasklet that checks for existing Reporting Date (cobDate) records in the RFT layer to prevent duplicate processing.
    
  - **Step 7:** masterTradeToTradeRFTStep Standard promotion from the Processing (ADS) layer to the Reporting (RFT) layer.
    
  - **Step 8:** masterTradeToRftTradeIfrs Migration of report-ready data into the Trade_ifrs specialized table.
    
  - **Step 9:** rftNettingEngineStep Executes core netting logic and transformations on the Trade_ifrs table.
    
  - **Step 10:** rftGVGEngineStep Two-part logic engine to update GVG Subtypes, Levelling, and Type fields across Trade_ifrs and GVG_Levelling tables.

**Phase 4: Final Aggregation Layer**

  - **Step 11:** createViewsFlowLinks Sets the vwFlowLink context using the RFT.dbo.Cash table for field-level linking.
    
  - **Step 12:** rftEngineStep The final consolidation engine:
    
    - **Staging:** Populates Quarterly/Monthly tables (TradeQ1_MS, TradeQ2_MSP, etc.).
    
    - **ItemProcessor Logic:** Joins quarterly data and migrates to RFT_Table_Summ.
    
    - **Field Mapping:** Updates IsNetted, Close_Balance, and Product_Code.
    
    - **Final Output:** Persists data into either RFT_Y_Summ (Yearly) or RFT_Q_Summ (Quarterly).

## 5. Resilience & Monitoring

**Restartability:** All chunk-based steps (2, 3, 5, 7, 8, 12) use the Spring Batch JobRepository to allow restarts from the last committed chunk.

**Fail-Fast Gates:** Step 6 acts as a critical quality gate, terminating the job if a Reporting Date conflict is detected.

**Scalability:** By adjusting the grid-size in the OpenShift ConfigMap, the application can scale horizontally across the cluster nodes.
