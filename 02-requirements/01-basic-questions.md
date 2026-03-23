Here is a structured overview of the system design and architecture questions, including **answers** based on the context of this Savings Account assignment:

### 1. Capacity, Scale & Throughput

| Question | Response |
|----------|----------|
| **What are the DAU (Daily Active Users) and MAU targets?** | Assuming a single-market MVP launch, targeting ~2,000 MAU and ~400 DAU, scaling to 20,000 MAU by end of Year 1. |
| **What are the Peak and Average TPS (Transactions Per Second)?** | Average API TPS will be very low (~2-10 TPS). Peak TPS might spike to ~100 TPS during end-of-month salary days or marketing pushes. CAMT processing TPS is handled asynchronously. |
| **What is the Read-to-Write ratio?** | Moderately Read-heavy (approx 5:1). Users log in to check balances/interest, and writes (deposits/interest/withdrawals) occur steadily. |
| **Are there seasonal traffic spikes (e.g., payday)?** | Yes, major spikes are expected on common salary days (e.g., 25th to 28th of the month) and at the start of new calendar years or tax seasons. |

### 2. Data Sizing & Storage

| Question | Response |
|----------|----------|
| **What is the expected database growth over 1 to 5 years?** | Profile+KYC data (~5KB/user) = 100MB per 20k users. Transactions (~1KB each) = 100MB/year. Total DB active size is very small (<2GB), making Aurora Serverless or DynamoDB highly cost-effective for active storage. |
| **What is the legal retention period for financial data?** | In compliance with standard European/local market regulations, financial and KYC records must be retained for 5 to 7 years after account closure. |
| **What is the daily volume/size of CAMT import files?** | A single daily CAMT.053 XML file containing hundreds of clearing transactions. Size is likely 2MB–10MB per day. |
| **What data belongs in Hot (Aurora) vs. Cold storage?** | Active accounts, balances, and recent transactions (last 12 months) stay in Hot storage. CAMT XML files and historical logs >1 year move to S3 Glacier (Roadmap C5). |

### 3. Availability & Latency

| Question | Response |
|----------|----------|
| **What is the target Availability SLA (e.g., 99.99%)?** | **99.9%** (approx 8.7 hours downtime/year) is sufficient for V1, utilizing AWS Multi-AZ deployments (Roadmap M17). |
| **What are our Disaster Recovery RTO and RPO targets?** | RPO (Recovery Point Objective) = 0 for financial ledgers (using synchronous multi-AZ replication). RTO (Recovery Time Objective) = <2 hours for complete RDS/Aurora failover. *(Note: Cross-region DR was explicitly marked out of scope for V1).* |
| **What is the acceptable latency budget for critical APIs?** | <200ms for read APIs (dashboard/balance). <2s for complex synchronous writes (e.g., IDV submission). |

### 4. Consistency & Financial Integrity

| Question | Response |
|----------|----------|
| **Where is Strong Consistency required?** | For the core ledger operations (deducting funds, crediting PYO, interest updates). This guarantees immediate visibility of a balance change to prevent double-spending or overdrafts. |
| **Where is Sequential Consistency required?** | For transaction history logs and audit ledgers. All observers must see chronological events (e.g., Deposit followed by Withdrawal) applied globally in the exact order they occurred. |
| **Where is Causal Consistency required?** | In compliance workflows and state machines. For example, if a KYC completion event *causes* an account to become ACTIVE, no part of the system should ever read the ACTIVE state without also being able to read the preceding KYC event. |
| **Where is Eventual Consistency acceptable?** | For customer read-only dashboards, search indices, and end-of-day analytics caching layers, where sub-second staleness is perfectly acceptable. |
| **How is exact idempotency enforced across async retries?** | By passing a unique `Idempotency-Key` (or transaction ID) in the API header, storing it in a Redis/DynamoDB lock or using DB unique constraints. This guarantees that SQS/DLQ retries never double-credit a deposit (Roadmap M22). |
| **Do we use Saga pattern or Event Sourcing for cross-service flows?** | We will start with standard distributed transactions utilizing the **Choreographed Saga pattern** (via EventBridge) for cross-service flows like Onboarding. Event Sourcing/CQRS is currently a "Could Have" (Roadmap C10) if audit requirements become stricter. |

### 5. Third-Party Integrations

| Question | Response |
|----------|----------|
| **What are the precise rate limits and SLAs of our KYC vendors?** | Assumes typical IDV vendor scale of ~10-20 async TPS with 99.9% API uptime. |
| **What is the exact fallback strategy during vendor outages?** | We will implement the Circuit Breaker pattern (Roadmap C8). If IDV/PEP screening is down, the system queues the customer application in a `PENDING_VERIFICATION` state, notifies the user via email, and processes asynchronously when the vendor recovers. |

### 6. Domain-Specific (Savings & Batch Jobs)

| Question | Response |
|----------|----------|
| **What is the maximum processing window for the nightly interest batch job?** | The batch job must complete between **00:00 and 04:00 local time** to ensure balances are fully updated and reconciled before morning traffic spikes starting at 07:00. |
| **How do we safely resume failed daily CAMT batch imports?** | The CAMT parsing Fargate job must track processed `MessageId` and `TxId` values in a state table. If the job crashes, the automated retry will skip already-committed transactions. |
| **How do we handle race conditions (e.g., deposit maturing right as an account is closed)?** | Implement **Pessimistic Locking** (`SELECT ... FOR UPDATE` in SQL) or **Optimistic Concurrency Control** (version numbers) on the Account record during all critical state transitions. |

### 7. Security, Compliance & Audit

| Question | Response |
|----------|----------|
| **What specific PII requires explicit field-level encryption?** | National ID numbers, tax identification numbers, and raw bank account details (IBANs) require field-level encryption (KMS). Basic Names/Addresses rely on standard AWS RDS/Aurora at-rest volume encryption (Roadmap M14). |
| **How do we mathematically or structurally guarantee audit log immutability?** | By writing highly sensitive compliance/audit events directly to an append-only ledger (AWS QLDB) or an AWS S3 bucket with Object Lock (WORM - Write Once Read Many) enabled (Roadmap M15). |
| **What is the KMS encryption key rotation policy?** | Automatic annual key rotation enabled within AWS KMS for standard keys, backed by manual rotation recovery playbooks for compromised key scenarios. |