## Architecture Decision Record: Data Staging Area for Batch Processing
Status: Accepted

1. Context
We need a robust way to handle ETL (Extract, Transform, Load) processes for large datasets. Without staging, direct ingestion causes several problems: 
Performance Issues: Complex transformations during extraction create high resource contention.
Low Reliability: Failure during the extraction/transformation process means the entire batch must be rerun, creating bottlenecks.
Audit/Compliance: Raw data is not stored, making it impossible to re-process data or trace data lineage in case of audit. 
Testbook
Testbook
 +4
2. Alternatives Considered
Alternative 1: Direct Load to Target (No Staging) - Lowers initial complexity but creates high risk of data inconsistency, slows down the production database, and makes re-processing difficult.
Alternative 2: In-Memory Processing - Too constrained by memory limits when handling large batch files.
Alternative 3: Dedicated External Staging Area (Accepted) - Provides a buffer between extraction and loading. 
Testbook
Testbook
 +4
3. Decision
We will adopt a dedicated staging area (using S3/Blob Storage/Temporary DB Schema) where data is stored in its raw format ("landing") before being transformed. 
Actian
Actian
Implementation Details:
Separation of Duties: Staging is strictly used for staging; it is not accessible to end-users.
Raw Data Retention: Raw files (CSV, JSON, Parquet) will be retained for a defined period (e.g., 7 days) to facilitate auditing and reprocessing.
Data Quality Checks: Data validation (null checks, schema verification) will occur after the data lands in staging but before the final transformation. 
Testbook
Testbook
 +4
4. Consequences
Positive
Resilience & Auditing: If the main processing job fails, the staging area holds the raw data, allowing for re-running the process without re-extracting from source systems.
Performance: Extraction from source systems happens in a quick "pull" operation, freeing up the data warehouse for queries.
Decoupling: Decouples the extractors from the transformers, allowing them to scale independently. 
Actian
Actian
 +4
Negative
Initial Complexity: Adds an extra step to the ETL/ELT pipeline.
Storage Costs: Requires additional storage capacity for the staging area.
Data Latency: Introducing an intermediary step adds latency to the overall data availability. 
Dremio
Dremio

 +2


 ## Usage of SFTP instead of directly quering source database

 1. Title
Usage of SFTP server for data file transfer instead of direct database queries. 
2. Status
Status: Accepted
Date: March 22, 2026
Author: [Your Name/Team Name]
3. Context
The project requires the regular transfer of large volumes of data from an external system to our internal data processing environment. The external system is managed by a third-party vendor. We need a reliable, secure, and performant method for this data exchange. 
The current options considered are:
Option 1: Allowing our system to directly query the source database via a dedicated API or direct connection.
Option 2: The external vendor periodically generating data files (e.g., CSV, JSON) and placing them on a secure, managed SFTP server for our system to retrieve. 
Security and data integrity are top priorities for this integration, and the data transfer is a scheduled batch process, not real-time. 
Merge.dev
Merge.dev
4. Decision
We have decided to use an SFTP server for data file transfers. The third-party vendor will place encrypted data files on a managed SFTP server, and our system will use secure automation to retrieve, validate, and process these files into our staging environment. 
Couchdrop
Couchdrop
 +1
5. Consequences
Positive Consequences
Enhanced Security: SFTP operates over the Secure Shell (SSH) protocol, providing strong encryption for both data in transit and authentication details, such as SSH keys, which is more secure than typical database username/password authentication. This helps us meet regulatory compliance requirements (e.g., GDPR, HIPAA).
Decoupling Systems: This approach decouples our internal data processing architecture from the external system's internal database structure. Changes to the source database schema will not directly break our integration, as long as the file format is maintained.
Performance and Scalability: SFTP is well-suited for transferring large, bulky files in one go (batch processing), which is more efficient for our use case than potentially complex direct database queries that might time out.
Network/Firewall Friendliness: SFTP typically uses a single port (default port 22), simplifying firewall configurations compared to the potential multiple ports or complex access rules needed for direct database connections.
Data Integrity: SFTP provides built-in data integrity checks using cryptographic hash functions, ensuring files are not tampered with during transfer. 
Reddit
Reddit
 +5
Negative Consequences
Near Real-Time Limitation: SFTP is less suitable for real-time data transfers due to the inherent delays of file generation and transfer schedules. This is acceptable for our specific batch processing requirements.
Additional Processing Layer: We need to implement a solution to pick up the files, validate their content, and load them into the database (e.g., using a file management solution or automation scripts). This introduces an extra layer of complexity compared to direct querying.
Automation Complexity: Automating SFTP processes can require manual scripting, though this can be mitigated using managed file transfer (MFT) solutions or cloud services like AWS Transfer Family.
Potential for Human Error: The manual process of generating and placing files by the vendor (if not fully automated on their end) introduces a higher potential for human errors compared to an automated API. 
Merge.dev
Merge.dev
 +4
6. Compliance
The use of SFTP helps ensure compliance with industry standards requiring encrypted and secure data transfer channels. The security measures, including SSH key authentication and data encryption, meet internal security policies. Data validation checks will be implemented in the ingestion process to ensure template compliance.


## Architecture Decision Record: Trade and Cash Data Partitioning
Status
Accepted
Context
Our Trade and Cash system is experiencing performance degradation due to rapid data growth. The main tables, storing trade details and cash balances, currently hold approximately 10 million records and are growing daily. 
Problems: Slow query performance on active data, long-running batch jobs, increased index maintenance time, and difficulty in archiving historical data.
Access Patterns: Queries frequently access recent trade/cash data (last 3-6 months), while historical data is rarely accessed but must be retained for auditing/compliance.
Goal: Improve read/write performance, enhance maintainability, and ensure scalability for future growth. 
GoCodeo AI
GoCodeo AI
 +4
Decision
We will implement Horizontal Partitioning (Range-based) on the trade and cash tables, using the Transaction Date (trade_date or cash_date) as the partitioning key. 
Partitioning Strategy
Partitioning Method: Range Partitioning.
Key: trade_date / cash_date (typically month-wise or year-wise, depending on volume).
Active Partitions: Current month/year partitions will be stored on fast storage (NVMe/SSD).
Historical Partitions: Past years' data will be moved to older, slower storage or archived to detached tables. 
Supporting Techniques
Vertical Partitioning: We will apply vertical partitioning to separate infrequently used large columns (e.g., audit logs, XML/JSON payload details) from the main, frequently accessed transaction table.
Archiving: Automated scripts will detach old partitions (> 2 years) to an archive database. 
EDB Postgres AI
EDB Postgres AI
 +2
Alternatives Considered
Vertical Partitioning Only: Rejected. It improves column-based query performance but does not solve the overall table growth issue (10 million rows).
Hash Partitioning: Rejected. While it provides even data distribution, it makes range-based queries (e.g., "get all trades in January") inefficient, as all partitions would need to be scanned.
No Partitioning (Scaling Up): Rejected. Upgrading to higher hardware specs is cost-prohibitive and provides only temporary relief. 
GoCodeo AI
GoCodeo AI
 +4
Consequences
Positive
Improved Query Performance: Queries filtering by date will only scan relevant partitions ("partition pruning"), reducing I/O.
Easier Maintenance: Dropping or archiving old data is as simple as dropping a partition rather than executing a slow DELETE query.
Scalability: Allows the system to handle millions more rows without significant performance dips. 
Codefinity
Codefinity
 +4
Negative/Risks
Query Complexity: Queries that do not include the partition key in the WHERE clause may be slower than before (scan all partitions).
Development Effort: Application code must ensure that queries include the date range (e.g., WHERE trade_date > '2025-01-01') to gain the benefits.
Join Constraints: Foreign keys must include the partitioning key, which can complicate schema design. 
Oracle - Ask Tom
Oracle - Ask Tom
 +4

### Based on a back-of-the-envelope estimation"
For 10 million records of trade and cash data, the total storage requirement is relatively modest, likely falling in the 10 GB to 100 GB range, depending on the complexity of the data and retention policies.
1. Assumptions
Total Records: 10 Million (
).
Average Record Size: 1 KB (including transaction ID, timestamp, user IDs, amount, currency, status, and metadata). While financial records are often smaller (e.g., 200–500 bytes), a 1 KB estimate provides a safe cushion for indexes and overhead.
Replication/Backup: Assume 3x replication for high availability (standard production practice). 
Medium
Medium
 +2
2. Calculation
Raw Data Size:

With 3x Replication/Backups:


With Indexing & Overhead:
Adding indexing, database logs, and transaction overhead (approx. 2x - 3x on top of raw), the storage could reasonably expand to 60 GB – 100 GB. 
3. Key Takeaways
Storage Feasibility: 10 million records is not considered "big data" in modern infrastructure. A single SQL or NoSQL node can handle this volume efficiently.
Data Type Factor: If the record includes substantial metadata (e.g., JSON blobs), the size might increase, but it is unlikely to exceed 200–300 GB unless storing raw packet data.
Retention: If this is 10 million records per day, the storage requirements grow by roughly 1 TB per month (with 3x replication). If this is a total, 30-100 GB is sufficient for 5+ years of retention. 


## 1. Title
## ADR-0XX: Adoption of OAuth2 Authentication and JWT Tokens
2. Status
Proposed/Accepted
3. Context & Problem Statement
Our application requires a secure, scalable authentication and authorization mechanism for distributed components (microfrontends, mobile, APIs). We need to support user identity verification and granular permission control (scopes) without incurring the performance penalty of frequent database lookups for session validation. 
Open edX
Open edX
 +3
4. Decision
We will use OAuth 2.0 with JWT access tokens for both authentication and authorization. 
WorkOS
WorkOS
Authentication: The authorization server will issue a signed JWT upon successful user login.
Authorization: Services will validate the JWT signature and enforce policies based on scopes and claims (e.g., user_id, role) embedded in the token.
JWT Handling: Clients will send JWTs in the Authorization: Bearer <token> header.
Key Management: Asymmetric signing (RS256 or similar) will be used to allow services to verify tokens without querying the identity provider directly. 
Read the Docs
Read the Docs
 +4
5. Mandatory JWT Claims
The following claims are required for all tokens to ensure security:
iss (Issuer): Verified to ensure the token came from our trusted IDP.
sub (Subject): Unique user identifier.
aud (Audience): Used to prevent token misuse across services.
exp (Expiration Time): Short-lived tokens to reduce breach risks.
jti (JWT ID): To prevent replay attacks. 
6. Alternatives Considered
Opaque Tokens: Rejected due to high latency (needs constant db checks) and low scalability.
Session Cookies (Traditional): Rejected for APIs; not suited for pure microservice architectures. 
Open edX
Open edX
 +4
7. Security Considerations
Secure Storage: JWTs will be stored in secure (HttpOnly, Secure, SameSite=Strict) cookies on frontend clients to mitigate Cross-Site Scripting (XSS) and Cross-Site Request Forgery (CSRF).
Short Lifespan: Access tokens will have short expiration times, with refresh tokens used to obtain new ones.
Validation: All services must validate the signature, expiration, and issuer of the token. 
Open edX
Open edX
 +4
8. Consequences
Positive: Stateless authentication enables high performance. Standardized protocols ease third-party integration.
Negative/Risk: Once issued, JWTs cannot be revoked before expiration unless a blacklist is implemented, which adds complexity. 
Open edX
Open edX

 +4





## ADR: Implementation of Asynchronous Report Generation with Distributed Locking
Status
Proposed / Accepted

Context
The current report generation process involves executing complex Stored Procedures (SP) in MS SQL Server, followed by file generation using the Apache POI framework.

Performance Bottleneck: The SP execution takes ~2 minutes, and Excel generation takes ~3 minutes.

Concurrency Issue: MS SQL Server encounters lock contention when multiple users request the same report simultaneously. If locks are not released within the timeout period, transactions are rolled back, resulting in a poor user experience and failed requests.

Resource Exhaustion: Keeping a synchronous HTTP connection open for 5+ minutes is inefficient and prone to timeouts at the Load Balancer/Gateway level.

Decision
We will decouple the report request from the generation process by implementing an Asynchronous Request-Response Pattern combined with a Time-Based Locking Mechanism at the application level.

Technical Components:
Non-Blocking API: The client receives an immediate 202 Accepted response with a unique reportId or status polling URL.

Java-Based Locking: Before executing the SP, the application will attempt to acquire a lock (e.g., using a ReentrantLock or a distributed lock like Redis/Hazelcast if in a clustered environment).

Timeout Logic: If the lock cannot be acquired within a specific threshold, the system will return a "Report in progress" message rather than allowing a database-level contention/rollback.

Decoupled Logic: The Stored Procedure execution and the Apache POI Excel writing logic will be executed in a separate background thread (e.g., @Async or a dedicated ExecutorService).

Consequences
Positive:

Database Stability: Prevents MS SQL Server transaction rollbacks and reduces overhead on the database engine by limiting concurrent executions of heavy SPs.

Improved UX: Users are not stuck on a loading screen. They can continue working while the report generates in the background.

Resilience: System timeouts (HTTP 504) are eliminated since the initial request is short-lived.

Negative/Neutral:

Increased Complexity: Requires implementing a status polling mechanism or a WebSocket/Notification system to alert the user when the file is ready.

State Management: The application must now track the state of reports (e.g., PENDING, PROCESSING, COMPLETED, FAILED) in a metadata table or cache.

Storage Requirements: Generated files must be temporarily stored (e.g., S3, Azure Blob, or a shared file system) until the user triggers the final download.
