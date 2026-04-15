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

## 6. Future Roadmap
The architecture is designed to evolve into an **Agentic AI** system. The structured data processed by Spring Batch will serve as the foundation for **Spring AI** and **LangChain4j** integrations, allowing for intelligent automated reporting and data analysis.
