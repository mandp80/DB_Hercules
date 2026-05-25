## Implement a Persistent Blocking Queue for Report Generation

### Context

The application provides a feature for users to generate complex financial reports. This process is initiated via an API endpoint that can receive requests to generate hundreds of reports at once. The core of this process involves executing a long-running, write-heavy stored procedure (Sp_GET_RFT_Template_Data_Hrcls_bq) for each report.

When multiple users submit these bulk requests concurrently, multiple application threads attempt to execute the stored procedure simultaneously. This has led to critical performance issues, including:

- Database Deadlocks: The stored procedure performs significant write operations, and concurrent executions frequently result in deadlocks, causing transactions to be rolled back.
- Transaction Timeouts: The long duration of the stored procedure (2-4 minutes) often exceeds transaction timeout limits.
- Poor User Experience: The API endpoint remains blocked until all processing is complete, leading to HTTP timeouts and an unresponsive UI.
- Lack of Resilience: If the application crashes mid-process, the user's request is lost, and there is no mechanism to resume the work.

Initial attempts to use in-memory locks (ReentrantLock) were insufficient as they do not work in a multi-instance (scaled) environment. A robust, durable, and scalable solution is required to serialize the execution of the stored procedure.

### Decision

We will implement a Persistent, Database-Backed Blocking Queue to manage and serialize the execution of report generation tasks. This refactors the architecture from a synchronous "process-now" model to an asynchronous "queue-and-process-later" model.

The implementation consists of three main components:

1. The Producer (PopulateTemplateServiceImpl):
	- The API endpoint (saveTemplateNamesAndCallSp) will no longer execute the stored procedure directly.
	- Its sole responsibility is to receive and validate the list of report requests.
	- It will use a transactional, bulk-processing strategy to create or update RftTemplateNameInputEntity records and insert corresponding tasks into a new report_generation_task table with a QUEUED status.
	- This makes the API call extremely fast, allowing it to return an immediate HTTP 200 OK to the client.
2. The Queue (report_generation_task table):
	- A new database table will act as the durable, persistent queue for all report generation requests.
	- Each row will represent a single task and will contain all necessary parameters and state information (e.g., status, created_at, error_message).
3. The Consumer (ReportTaskProcessorService):
	- A new background service will be created to process tasks from the queue.
	- It will use a "Push-to-Pull" (Poke) pattern for efficiency in a single-instance deployment. An in-memory TaskSignalService will be used to wake the consumer thread only when new tasks are available, avoiding continuous database polling during idle periods.
	- The consumer's worker thread will fetch tasks from the database in batches. For each task, it will: a.  Update the task's status to SP_PROCESSING in a separate, immediate transaction for UI visibility. b.  Execute the long-running stored procedure. c.  Upon completion, update the task's status to SP_COMPLETED and hand it off to an asynchronous ExcelGenerationService.
	- This ensures that the execution of the stored procedure is serialized, processing only one at a time.

### Consequences

Positive

- Deadlocks Eliminated: By processing one stored procedure at a time, database deadlocks related to this operation are completely resolved.
- Improved API Responsiveness: The API is now non-blocking and returns immediately, significantly improving user experience and system stability.
- Durability and Resilience: Report requests are persisted in the database. If the application restarts, no work is lost, and processing can resume automatically.
- Observability and Auditing: The state of every report generation task is now explicitly tracked in the database, making it easy to monitor progress, debug failures, and gather metrics on processing times.
- Improved Code Structure: The logic is cleanly separated into distinct services for producing tasks, processing them, and generating files, adhering to the Single Responsibility Principle.

Negative

- Increased Latency for Single Reports: A single, isolated report request will now have a slightly higher end-to-end latency due to the queuing mechanism. This is a necessary trade-off for overall system stability and throughput.
- Increased Architectural Complexity: The system now includes additional components (new tables, entities, and services) to manage the queue, which adds to the overall complexity of the codebase.
- Dependency on Database Performance: The throughput of the system is tied to the performance of the report_generation_task table. Proper indexing on the status and createdAt columns is critical.

Alternatives Considered

1. In-Memory Blocking Queue
	- Description: Use a standard Java BlockingQueue within a singleton service to serialize tasks.
	- Pros: Very fast and simple to implement for a single application instance.
	- Cons: Not scalable. In a multi-instance environment, each application instance would have its own separate queue, re-introducing the original concurrency problem. It also lacks durability; all pending tasks are lost on application restart. This was discarded as it does not meet resilience and scalability requirements.
2. Optimistic Locking
	- Description: Use a @Version column on the target entities to prevent concurrent modifications. Threads would read the data, and upon trying to commit, the transaction would fail if the version number has changed.
	- Pros: Higher concurrency for read operations compared to pessimistic locking.
	- Cons: Not suitable for this specific problem. The primary issue is contention on a shared resource (the stored procedure's write operations), not just on a single entity row. Optimistic locking would lead to frequent OptimisticLockExceptions and complex retry logic, effectively creating a "spin-lock" at the application layer, which is less efficient than a queue.
3. External Message Broker (e.g., RabbitMQ, Kafka)
	- Description: Use a dedicated message broker to manage the task queue. The producer sends messages to a queue, and one or more consumers listen for and process these messages.
	- Pros: The "gold standard" for distributed task queuing. Highly scalable, resilient, and provides advanced features like dead-letter queues and routing.
	- Cons: Introduces a significant new technology to the stack, increasing operational overhead, infrastructure costs, and development complexity. For the current scale and requirements, this was deemed an over-engineered solution. The database-as-a-queue pattern provides sufficient functionality with the existing technology stack.
