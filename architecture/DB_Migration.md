## 1.1. Objective of the Migration
The primary objective of this migration is to address the following key issues observed in the current setup:

Date Security: - Input files are currently shared over email, posing a data security.
Manual Processes: - Multiple tasks are performed manually, including IT team accessing the production environment to execute QE tasks.
Named user Dependency: - Upstream applications require named user logins to extract data, reducing efficiency and increasing dependency.
Performance Limitations: - The existing standalone Cerberus application suffers from performance issues and is unable to handle concurrent QE requests effectively.
Lack of Audit Trail: - There is no proper audit trail available from source to target making it difficult to trace data lineage and changes.
Complex Excel Templates: - Users rely on complex Excel templates to view Cerberus data and prepare files, which are later reloaded into the system.
Data Issues: - Inconsistent and unreliable data updates have been observed in the current system.
IT team Dependency: - Data updates are manually performed by IT team using SQL Server Management Studio (SSMS) scripts due to the absence of a proper UI interface.
Technology Limitations: - The current .NET based windows application and SQL Server database are not scalable and have become a bottleneck for application enhancement and migration to GCP. These technologies are not aligned with the long-term data strategy.

## 1.2. Benefits of the Migration 
Enhanced Data Security: - Eliminate email-based file sharing by adopting secure, centralized data handling methods.
Efficiency: - Fast and easy deployment, CI/CD, can be onboarded.

Risk & Security: - Comply with all engineering and security guidelines ,2FA authentication, Easy to apply Quality and sustainability, Risk & Compliance principles, Unit test, BDD, Sonar, X-Ray, VERACODE etc 

Reduce Manual Intervention: - Automate QE tasks and operational processes, minimizing the need for IT teams to access production environment directly.
Process monitoring: - 
Ensure Process Continuity - Detect failures or Interruptions in automated jobs and scheduled tasks.
Improve Operational Visibility - Provide a clear view of the status and performance of key processes and components.
Enable early issue Detection - Catch error or delays before they impact Business outcomes.
Facilitate Troubleshooting - Logs and alerts from monitored processes help teams identify root causes quickly.
Support Compliance and Auditing -Maintain logs and status records for audit trails and compliance requirements.
Improved System Performance: - Migrating away from a standalone .NET windows application improves scalability and system responsiveness, especially under concurrent loads.
Centralized Authentication and Authorization: - Integrated with CIDP to manage users and permissions centrally, removing dependency on local application-level authentication.
Auditability and Compliance: - Enables full audit trails from source to target, supporting traceability and compliance with regulatory standards.
Elimination of Complex Excel Workflows: - Replaces Excel-based manual template with structured, UI-driven solutions, reducing error and improving user efficiency.
Improved Data Integrity: - Eliminates directly database updates via SSMS by providing proper UI interfaces and validation controls.
Future Ready Technology Stack: - Aligns with organization's GCP and data strategy by replacing legacy technologies like .NET and MS Office.
Better User Experience: -New application interfaces offer more intuitive and responsive designs for users, improving overall usability.
Operational Efficiency: - Streamlined workflows, automated processes and improved visibility contribute to faster, more reliable operations.
Tasks Controls: - Implemented to manage, coordinate and validate the execution of individual tasks within a system workflow. This control ensure that tasks are executed in the correct sequence.
