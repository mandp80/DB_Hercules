### Trade and Cash Data Partitioning
- **Status:** Accepted
- **Context:** Our Trade and Cash system is experiencing performance degradation due to rapid data growth. The main tables, storing trade details and cash balances, currently hold approximately 10 million records and are growing daily. 
Problems: Slow query performance on active data, long-running batch jobs, increased index maintenance time, and difficulty in archiving historical data.
Access Patterns: Queries frequently access recent trade/cash data (last 3-6 months), while historical data is rarely accessed but must be retained for auditing/compliance.
- **Goal:** Improve read/write performance, enhance maintainability, and ensure scalability for future growth. 
- **Decision:** We will implement Horizontal Partitioning (Range-based) on the trade and cash tables, using the Transaction Date (trade_date or cash_date) as the partitioning key. 
  - **Partitioning Strategy:**
    Partitioning Method: Range Partitioning.
    Key: trade_date / cash_date (typically month-wise or year-wise, depending on volume).
    Active Partitions: Current month/year partitions will be stored on fast storage (NVMe/SSD).
    Historical Partitions: Past years' data will be moved to older, slower storage or archived to detached tables. 
  - **Supporting Techniques:**
    Vertical Partitioning: We will apply vertical partitioning to separate infrequently used large columns (e.g., audit logs, XML/JSON payload details) from the main, frequently accessed transaction table.
    Archiving: Automated scripts will detach old partitions (> 2 years) to an archive database. 
- **Alternatives Considered:**
  - Vertical Partitioning Only: Rejected. It improves column-based query performance but does not solve the overall table growth issue (10 million rows).
  - Hash Partitioning: Rejected. While it provides even data distribution, it makes range-based queries (e.g., "get all trades in January") inefficient, as all partitions would need to be scanned.
  - No Partitioning (Scaling Up): Rejected. Upgrading to higher hardware specs is cost-prohibitive and provides only temporary relief. 

- **Consequences:**
  - **Positive:**
    - Improved Query Performance: Queries filtering by date will only scan relevant partitions ("partition pruning"), reducing I/O.
    - Easier Maintenance: Dropping or archiving old data is as simple as dropping a partition rather than executing a slow DELETE query.
    - Scalability: Allows the system to handle millions more rows without significant performance dips. 

  - **Negative/Risks:**
    - Query Complexity: Queries that do not include the partition key in the WHERE clause may be slower than before (scan all partitions).
    - Development Effort: Application code must ensure that queries include the date range (e.g., WHERE trade_date > '2025-01-01') to gain the benefits.
    - Join Constraints: Foreign keys must include the partitioning key, which can complicate schema design. 
