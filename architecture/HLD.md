# High-Level Design (HLD): DB_Hercules

## 1. Executive Summary
**DB_Hercules** is a cloud-native architectural blueprint designed to modernize legacy monolithic banking reporting systems. The platform transitions complex, high-volume data workflows into a microservices ecosystem deployed on an on-premises **OpenShift** cluster. The core processing engine is built on **Spring Batch**, ensuring robust, transactional, and restartable data ingestion.

## 2. Architectural Overview
The architecture is designed for reliable batch processing of financial data, leveraging the container orchestration of OpenShift and the modularity of Spring Boot.

### 2.1 Core Components
* **Frontend**: A modern **ReactJs** Single Page Application (SPA) for monitoring and management.
* **API Gateway**: Handles request routing, secure authentication, and load balancing.
* **Processing Engine (Spring Batch)**: The heart of the system, managing the lifecycle of data ingestion jobs.
* **Microservices**: Independent **Spring Boot** services for file management and report generation.
* **Infrastructure**: **OpenShift (Kubernetes)** for managing the containerized application lifecycle.

---

## 3. Data Ingestion Architecture
The system employs **Spring Batch** to manage the intake of diverse financial datasets. This choice provides built-in support for chunk-based processing, which is essential for handling large-scale banking data efficiently.

### 3.1 Ingestion Protocols
| Data Type | Source System | Protocol | Ingestion Strategy |
| :--- | :--- | :--- | :--- |
| **Cash Data** | ACS - Cash Ledger | **SFTP** | Spring Batch SQLServerBulkCopy loads CSV files received via secure SFTP stream. |
| **Trade Data** | FDW - Trade Booking | **JDBC** | `JdbcPagingItemReader` executes direct queries for high-volume, partitioned data extraction. |
| **FX Rates** | Mercury | **REST** | Specialized Tasklet calls the **Mercury KDB API** for time-series rate retrieval. |

### 3.2 Resilience & Reliability
By utilizing Spring Batch instead of standard messaging, the system gains:
* **Restartability**: Failed jobs can be resumed from the last successful chunk, preventing data duplication.
* **Transaction Management**: Ensures that data is only committed to the destination once a full chunk is processed successfully.
* **Skip/Retry Logic**: Robust handling of malformed records or temporary network timeouts during SFTP or API calls.

---

## 4. DevOps & Infrastructure as Code (IaC)
The project automates both the infrastructure and application layers to ensure environment parity.

### 4.1 Provisioning & Orchestration
* **Terraform**: Provisions the "Static Infrastructure" (OpenShift Namespaces, RBAC, Service Accounts, and Persistent Volumes).
* **Helm**: Orchestrates the "Application Layer" (Deployments, Services, Routes, and ConfigMaps).

### 4.2 Automated CI/CD
The **Jenkins** pipeline uses a `Jenkinsfile` to execute:
1. **Terraform Apply (Yet to be developed)**: Validates and provisions the infrastructure.
2. **Build & Test**: Compiles the Spring Boot applications.
3. **Deployment**: Uses Helm to push containerized batch jobs and services to the OpenShift cluster.

---

## 5. Security & Compliance
As an investment banking solution, DB_Hercules adheres to strict security standards:
* **Credential Isolation**: SFTP and Database credentials are encrypted and injected via OpenShift Secrets.
* **Secure Transport**: All external communications (SFTP and Mercury REST API) are performed over encrypted channels (TLS/SSH).
* **Audit Logging**: Spring Batch `JobRepository` maintains a comprehensive audit trail of every data ingestion event.

---

## 6. Future Roadmap & Strategic Enhancements

The following objectives outline the transition from the current modernization phase to a fully optimized, enterprise-grade reporting ecosystem.

### 6.1 Transition to Native System Integration
Currently, secondary datasets—including **Bond ISINs, Levelling, Product Codes, Opening Balances, and UBR Hierarchies**—are ingested via a legacy CSV/Excel import strategy. 
* **Enhancement:** Shift from manual file dependencies to direct system-to-system integration.
* **Objective:** Implement data sourcing and ingestion using **bank-standard communication protocols**. This will eliminate file-handling risks and ensure real-time data synchronization.

### 6.2 Modernization of the Adjustment & Reporting Layer
The final stage of report generation and manual data adjustment is currently handled through Excel-based macros. These are prone to errors and face performance bottlenecks with large datasets.
* **Enhancement:** Develop a centralized, **UI-based adjustment module** within the ReactJs dashboard.
* **Objective:** Replace manual macros with a robust interface that supports high-volume data validation and automated report generation.

### 6.3 Optimization of Trade Data Ingestion Throughput
The current ingestion window for Trade data is approximately **2 hours**, utilizing 400 connection pools and a 1000 record chunk size.
* **Enhancement:** Conduct performance tuning of the Spring Batch orchestration.
* **Objective:** Significantly reduce the ingestion window by implementing **multi-threaded step partitioning** and tuning database concurrency to balance chunk size against connection overhead.

### 6.4 Strategic Overview: Current vs. Target State

| Feature | Current State | Target State (Roadmap) |
| :--- | :--- | :--- |
| **Data Sourcing** | Manual CSV/Excel Imports | Direct Protocol Integration |
| **Adjustments** | Excel Macros | React-based UI Module |
| **Throughput** | 2-Hour Ingestion Window | Optimized Parallel Processing |
| **Architecture** | File-centric Ingestion | System-centric Integration |
| **Intelligence** | Procedural Processing | Agentic AI (Spring AI/LangChain4j) |
