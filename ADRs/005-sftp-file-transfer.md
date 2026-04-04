### Usage of SFTP instead of directly quering source database

- **Status:** Accepted
- **Context:** The project requires the regular transfer of large volumes of data from an external system to our internal data processing environment. The external system is managed by a third-party vendor. We need a reliable, secure, and performant method for this data exchange. 
The current options considered are:
  - **Option 1:** Allowing our system to directly query the source database via a dedicated API or direct connection.
  - **Option 2:** The external vendor periodically generating data files (e.g., CSV, JSON) and placing them on a secure, managed SFTP server for our system to retrieve. 
Security and data integrity are top priorities for this integration, and the data transfer is a scheduled batch process, not real-time. 

- **Decision:** We have decided to use an SFTP server for data file transfers. The third-party vendor will place encrypted data files on a managed SFTP server, and our system will use secure automation to retrieve, validate, and process these files into our staging environment. 
- **Consequences:**
  - **Positive Consequences:**
    - **Enhanced Security:** SFTP operates over the Secure Shell (SSH) protocol, providing strong encryption for both data in transit and authentication details, such as SSH keys, which is more secure than typical database username/password authentication. This helps us meet regulatory compliance requirements (e.g., GDPR, HIPAA).
    - **Decoupling Systems:** This approach decouples our internal data processing architecture from the external system's internal database structure. Changes to the source database schema will not directly break our integration, as long as the file format is maintained.
    - **Performance and Scalability:** SFTP is well-suited for transferring large, bulky files in one go (batch processing), which is more efficient for our use case than potentially complex direct database queries that might time out.
    - **Network/Firewall Friendliness:** SFTP typically uses a single port (default port 22), simplifying firewall configurations compared to the potential multiple ports or complex access rules needed for direct database connections.
    - **Data Integrity:** SFTP provides built-in data integrity checks using cryptographic hash functions, ensuring files are not tampered with during transfer. 
  - **Negative Consequences:**
    - **Near Real-Time Limitation:** SFTP is less suitable for real-time data transfers due to the inherent delays of file generation and transfer schedules. This is acceptable for our specific batch processing requirements.
    - **Additional Processing Layer:** We need to implement a solution to pick up the files, validate their content, and load them into the database (e.g., using a file management solution or automation scripts). This introduces an extra layer of complexity compared to direct querying.
    - **Automation Complexity:** Automating SFTP processes can require manual scripting, though this can be mitigated using managed file transfer (MFT) solutions or cloud services like AWS Transfer Family.
    - **Potential for Human Error:** The manual process of generating and placing files by the vendor (if not fully automated on their end) introduces a higher potential for human errors compared to an automated API. 
- **Compliance:** The use of SFTP helps ensure compliance with industry standards requiring encrypted and secure data transfer channels. The security measures, including SSH key authentication and data encryption, meet internal security policies. Data validation checks will be implemented in the ingestion process to ensure template compliance.
