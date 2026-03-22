## Caching Reference Data (FX Rates/ISINs) in Spring Batch

1. Context
The application processes large volumes of transactions requiring enrichment with volatile FX rates (hourly/daily updates) and static/semi-static Bond ISIN master data. Querying the database for each record causes high I/O latency, database strain, and significant batch execution delays.
2. Decision
We will implement an in-memory caching strategy within the Spring Batch step (e.g., in the ItemReader or ItemProcessor) to store reference data. 
Caching Technology: Caffeine Cache (due to high performance and efficiency).
Cache Scope: Job level (loaded once at the start of a Step or Job).
Data Type:
Bond ISINs: Cache permanently for the session (long-lived).
FX Rates: Cache with a time-based expiration (e.g., expire after 1 hour or 10 minutes) or load fresh per job run. 
The R Journal
The R Journal
 +4
3. Consequences
Positive:
Improved Performance: Sub-millisecond data retrieval instead of database network calls.
Reduced I/O: Drastically lowers database load.
Consistency: Data is consistent throughout the batch processing step.
Negative/Risks:
Memory Usage: High memory utilization, risk of OutOfMemoryError if the cache size is not controlled (must use loading cache with maximum size).
Stale Data: Risks using slightly outdated FX rates if not configured properly. 
4. Implementation Plan
Implement CacheManager to configure caching constraints.
Pre-load static ISIN data using StepExecutionListener#beforeStep.
Implement lazy-loading for FX rates with a short TTL (Time-To-Live). 
5. Alternatives Considered
Database Joins: Too slow for millions of records.
Distributed Cache (e.g., Redis): Added network latency; unnecessary for local batch processing constraints. 
