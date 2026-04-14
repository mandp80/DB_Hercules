## Hercules - High Level Design (HLD)

![Alt text](images/Hercules-HLD.drawio.png)

This design illustrates a modern, containerized approach to banking data reporting, transitioning from a monolithic system to a scalable microservices architecture.

### Core Infrastructure: OpenShift Cluster:

The entire application is hosted on an on-premises OpenShift Cluster, which provides the orchestration layer for the microservices as well as Data Ingestion Batch Jobs.

- **Scalability:** The cluster manages the lifecycle of the Spring Boot containers, allowing for horizontal scaling based on the reporting load.

- **Management:** The plan is to manage Infrastructure via Terraform, while application deployments are already handled through Helm charts.


### Data Ingestion & Processing Layer:

This layer is responsible for the intake of raw financial data into the system. A specialized processes created that handle the scheduled or event-driven intake of data files.

  - **Cash data** is in csv format pushed to SFTP location along with control file containing file hash code to perform data integrity check. Data Ingestion job pulls these files connecting to SFTP location, process them and store them to database.

  - FDW system holds **Trade data** in their Oracle database system and exposed read-only database schema for the consumer to fetch data using SQL queries via TLS. Hercules connects to FDW database through TCPS connection and queries relevent trade tables.

  - Mercury is Deutsche Bank standard source system for global **FX rates** and it exposes it's service via KDB API.

  - In-memory cache has been used to pre-load the FX rates and other reference data for Trade and Cash data processing job to improve the performance.
 
### Application Services Layer:

The business logic is decomposed into specific microservices built with Spring Boot.

  - **File Upload/Download Service:** A dedicated service managing the storage and retrieval of reporting documents. It interacts with the persistent storage layer of the cluster to ensure data durability.

  - **Reporting Microservices:** (Implied by context) These services process the ingested data to generate the compliance and business reports required by the Bank to submit it.

### User Interaction Layer (Frontend):

  - **ReactJs SPA:** The user interface is a Single Page Application (SPA) built with React, providing a responsive and interactive dashboard for bank users to monitor reporting status and manage files.

  - **API Gateway/Route:** The frontend communicates with the internal microservices via OpenShift Routes and an integrated API gateway that handles authentication and request routing.

### Continuous Integration & Delivery (CI/CD):

The design is supported by a robust automation pipeline:

  - **Jenkins:** Serves as the primary automation server.

  - **Jenkinsfiles:** Define the pipeline-as-code, including stages for building the Spring Boot JARs, running tests, and pushing images to the registry.

  - **Helm:** Automates the deployment of complex Kubernetes objects within the OpenShift environment.
