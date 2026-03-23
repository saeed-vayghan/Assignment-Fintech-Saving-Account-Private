# Payments Team: Architecture Explanation & Nuances

This document explains the architectural decisions behind dividing the Payments team's domain into the specific microservices exactly as defined in the `03-payments.md` profile.

### The 4 Microservices:
1. `CAMT Ingestion Pipeline`
2. `Tax Engine`
3. `Payout Job Orchestrator`
4. `Regulatory Reporter`

---

## 1. Are they Serverless?
**Yes, entirely.** The Payments team is the heaviest user of asynchronous Serverless orchestration across the bank. They rely on **AWS Lambda** triggered by Cron jobs (EventBridge Scheduler), S3 Object Creates, and SQS Queues, storing their temporary idempotency state in **Amazon DynamoDB**. 

Because this team has no human-facing UIs (no 200ms real-time API latency requirements), Lambda's ability to fan-out to 10,000 parallel workers to crunch end-of-month data in seconds is a massive advantage compared to running permanent batch-processing EC2 servers.

---

## 2. Why exactly these 4 boundaries? (The Challenge)

Let's challenge if we have too many or too few services by evaluating their Domain-Driven Design (DDD) boundaries and failure domains.

### 1. CAMT Ingestion Pipeline
* **Why it exists:** Strictly event-driven and decoupled. When an external partner bank uploads a file via SFTP, AWS Transfer Family places it directly into S3, triggering an `ObjectCreated` event that wakes up this pipeline to parse the massive XML files (ISO 20022 CAMT.053).
* **Challenge:** *Could we merge this with the actual Ledger?* Absolutely not. CAMT files are notoriously messy and require heavy XML parsing memory. An XML parsing failure or a corrupted un-schema'd file should trigger a DLQ alert, but it has no business disrupting the core ledger synchronous APIs.

### 2. Tax Engine
* **Why it exists:** Automatically computes mandatory withholding taxes before payouts occur.
* **Challenge:** *Why separate Tax into Payments while Interest is in Deposits?* **Regulatory Velocity.** Tax laws (withholding rates, deduction caps) change frequently depending on local government legislation. Interest logic, however, is a static mathematical financial product tightly coupled to the Core Ledger. By separating the Tax Engine, the FinOps team can deploy rapid updates to tax withholding percentages without risking regressions in the strictly mathematical core interest-compounding logic owned by the Deposits team.

### 3. Payout Job Orchestrator

### 4. Payout Job Orchestrator
* **Why it exists:** Manages the state machine for processing massive payout batches for matured accounts or canceled accounts. It guarantees that if a batch of 10,000 payouts crashes at #5,000, it can resume perfectly without double-paying anyone (via strict DynamoDB idempotency locks).
* **Challenge:** *Why a dedicated Orchestrator?* Because moving money *out* of the bank is the highest risk action in the architecture. This logic must wrap AWS Step Functions or SQS DLQs to handle transient network errors to the underlying Central Bank payout APIs. Mixing "how much money do we owe?" (Interest Engine) with "how do we securely transmit it?" (Payout Orchestrator) violates the Single Responsibility Principle.

### 4. Regulatory Reporter
* **Why it exists:** Executes scheduled CRON jobs to aggregate core ledger transaction metadata into standardized XML reports for the Central Banking Authority.
* **Challenge:** *Why not let the Platform team handle reporting?* **Domain Isolation.** Setting up XML formats and data mappings for the central bank requires deep financial domain knowledge. The Platform team is generic compute/infrastructure. By placing the Regulatory Reporter in Payments, FinOps engineers can rapidly deploy new schema rules without dragging generic infrastructure specialists into financial logic.

---

## Conclusion
The Payments architecture is a textbook example of **Batch-Processing Decoupling**. By separating Ingestion, Computation (Interest/Tax), and Transmission (Orchestration), the bank can safely re-trigger or isolate failures. If the Tax Engine fails on December 31st, the orchestration queue will safely halt, allowing developers to patch the codebase and resume the SQS queue without dropping a single payout.
