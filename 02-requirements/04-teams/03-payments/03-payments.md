# Team 3: Payments & Accounting

### **Purpose**:
Ingest, parse, and process massive daily batch files (CAMT.053) for incoming deposits (M8), and perform heavy algorithmic computations for daily end-of-month interest (M9) and withholding taxes (M10). Finally, manage the orchestration for outgoing payouts (M11) and financial regulatory reporting (M30).

### **UI Applications**:
No. This is a purely backend, asynchronous data-processing team.

### **User-Facing Service**:
No. Customers never interact with Payment services directly.

### **Authentication / Authorization Mechanisms**:
*(See `06-auth-service.md` for Centralized Issuance / Decentralized AuthZ strategy)*
- **Payments Engine → Deposits Ledger:** Authorized outbound via `OAuth-M2M` or strict `AWS-IAM` execution roles.
- **Async Queues / Scheduled Jobs:** Authorized purely via `AWS-IAM` resource policies inside the private VPC subnet.
- **Partner Bank SFTP (CAMT Files):** Authenticated inbound via `SSH-Key` pairs managed by AWS Transfer Family.

### **Microservices**:
  1. `CAMT Ingestion pipeline`: An event-driven Lambda triggered automatically by S3 `ObjectCreated` events when partner banks cleanly drop XML files via AWS Transfer Family. Parses ISO 20022 rows and explicitly creates `PYI` entries (M8).
  2. `Tax Engine`: Automatically computes mandatory withholding taxes (M10) before payouts occur.
  3. `Payout Job Orchestrator`: Manages the state machine for processing massive payout batches for matured accounts or canceled accounts (M11), strictly enforcing idempotency across all write operations (M22).
  4. `Regulatory Reporter`: A scheduled CRON job that pulls aggregated financial data from the core ledger to generate mandatory M30 end-of-month central bank transaction reports.

### **Databases & Storage Access**:

| Database / Storage | Type | Access Level | Purpose / Justification |
|--------------------|------|--------------|-------------------------|
| **Amazon DynamoDB** (Payment Locks) | NoSQL State Store | **Read / Write** (Owned) | **Idempotency Locks:** Conditional writes guarantee we never process the exact same payout job twice (M22). |
| **Amazon S3** (Settlement & Archive) | Object Storage | **Read / Write** (Owned) | **File Archive:** Cheap, secure storage for incoming XML (CAMT.053) files and outgoing Central Bank Regulatory Reports. |
| **Amazon Aurora (PostgreSQL)** (Deposits Ledger) | Relational SQL | **Write** (via Internal API) | **Cross-Team Dependency:** Payments calculates the math but utilizes secure internal APIs to commit final entries to the core ledger owned by **Team 2 (Deposits)**. |
| **ELK Stack** (Central Logging) | Search Index | **Write** (via Platform API) | **Cross-Team Dependency:** Regulated batch execution logs and error alerts are pushed to the central observability cluster owned by **Team 4 (Platform)**. |

### **Caching Layer**:
None required for asynchronous batch processing.

### **Message Queue / Async Comm**:
Heavily async.
- Uses **S3 Event Notifications** for triggering real-time ingestion of clearing files.
- Uses `AWS EventBridge` to subscribe to the `AccountMatured` domain event published by the Deposits team, triggering the automated Tax and Payout pipeline.
- Uses `AWS SQS` for batching payout tasks and handling automated retry / Dead Letter Queues (DLQ) for temporary failed operations (S3).

### **External Systems**:
Central/Partner Banks (via SFTP protocol for pushing CAMT files). The Financial Authority (for sending transaction reports).

### **Third-Party Services**:
None directly (AWS Transfer Family completely manages the SFTP hosting).

### **Other Team Interactions**:
  - **Platform Team**: Leverages their infrastructure for DLQs and relies on their secure networking (VPC/SFTP).
  - **Deposits Team**: Owns the core ledger and the overnight interest calculation. Payments acts strictly as an orchestration layer for taxes and physical fiat payouts based on events emitted by Deposits.

### **Edge Cases & Failure Scenarios**:

| Category | Scenario | Immediate Strategy / Impact | HITL Action / Threshold |
|----------|----------|-----------------------------|-------------------------|
| **Retryable** | **SFTP Server Unavailable:** Cannot pull daily CAMT file from central bank. | Cron job exponentially retries every 15 mins. | **FinOps (Major):** If delayed >4 hours, manually call central bank to verify system status. |
| **Retryable** | **DynamoDB Idempotency Timeout:** Payout job crashes mid-processing. | SQS visibility timeout expires, message returns to queue. | **FinOps (Moderate):** If sent to DLQ (S3), investigate root cause before manual retry. |
| **Moderate** | **CAMT Parse Error / Corrupted XML:** Partner network uploads malformed file. | Parsing halts, file moved to dead-letter S3 bucket. | **FinOps:** Manually fixes file encoding/syntax and re-uploads to SFTP for processing. |
| **Moderate** | **Unmatched PYI:** CAMT contains deposit for a non-existent IBAN. | Funds held in a suspense aggregate account. | **Support:** Manually contacts sender or initiates return transaction. |
| **Major** | **Unexpected Target Bank Holiday:** EOM payout jobs fail on destination clearing. | Payout orchestration queue stalls out. | **FinOps:** Pauses SQS queue and triggers comms team to notify users of 1-day delay. |
| **Major** | **Tax Engine Logic Bug:** Code regression causes 0% withholding tax. | Risk of severe regulatory penalties. | **Accounting:** Halt payouts immediately, run manual tax correction scripts before resuming. |

### **Logs and Metrics**:
Extremely detailed file processing metrics (files received, rows parsed, failed matches), data-integrity/reconciliation reports, and structured transaction reports generated explicitly for the central bank/regulator (M30).

### **API Exposed**:
  - Internal REST APIs (to manually trigger or retry failed batch jobs/DLQs)
  - Asynchronous event consumers (listening to queues)

### **Roadmap & MoSCoW Prioritization References**:
#### **Must Have (MVP)**
- `M8`: CAMT file import from SFTP (parse, match, create PYI)
- `M10`: Tax calculation engine (withholding tax)
- `M11`: Payout processing job (PYO for matured/withdrawn, CAN for cancellations)
- `M22`: Idempotency on all write operations
- `M30`: Regulatory reporting: transaction reports to central bank / financial authority

#### **Should Have**
- `S3`: DLQ handling & automated retry for failed payments
