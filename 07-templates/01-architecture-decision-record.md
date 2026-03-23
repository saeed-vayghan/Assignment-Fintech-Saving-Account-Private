# ADR 001: Database Selection for Core Ledger Service
 
 **Date:** 2026-03-23
 **Status:** Accepted
 **Service:** Core Ledger API (Team 2)
 
 ---
 
 ## 1. Context and Problem Statement
 The Core Ledger is the central financial source of truth for the bank. It must record all deposit and withdrawal events (`PYI` and `PYO`) with infallible mathematical accuracy. We must select a primary operational datastore that balances strict **ACID compliance** (Atomicity, Consistency, Isolation, Durability) against the need for **cloud-native horizontal scalability**.
 
 ## 2. Decision Drivers
 - **Data Integrity:** Absolutely zero tolerance for financial anomalies (e.g. double-spending) or eventual consistency in ledger balances.
 - **Reporting & Auditing:** The business requires generating complex end-of-month regulatory statements involving deep relational joins across users, accounts, and transactions.
 - **High Availability:** The system must seamlessly survive an AWS Availability Zone (AZ) failure with an RPO (Recovery Point Objective) near zero.
 
 ## 3. Considered Options
 1. **Amazon DynamoDB (NoSQL)**
    - *Pros:* Practically limitless scaling, single-digit millisecond latency, fully serverless operational model.
    - *Cons:* Lacks native relational constraints (like foreign keys). Structuring multi-table ACID transactions is complex and expensive. Lacks native SQL, making monthly reporting highly inefficient.
 2. **Amazon Aurora PostgreSQL (Relational) [SELECTED]**
    - *Pros:* Native mathematical ACID guarantees and row-level locking. Standard SQL simplifies complex relational reporting. Horizontally scales read replicas and synchronously replicates across AZs for high availability. 
    - *Cons:* Write-scaling is bound to the single primary instance. Subject to connection exhaustion under high serverless load.
 
 ## 4. Decision
 We will build the Core Ledger Service utilizing a **Multi-AZ Amazon Aurora PostgreSQL cluster** as the primary operational database.
 
 ## 5. Consequences
 ### Positive Outcomes
 - **Financial Accuracy:** We can utilize native database constraints (e.g., `CHECK (balance >= 0)`) and pessimistic locking (`SELECT ... FOR UPDATE`) to strictly prevent balance overdrafts without overly complex application logic.
 - **Reporting Velocity:** Complex financial reporting is handled natively by the database engine via standard SQL aggregations.
 
 ### Negative Outcomes & Mitigations
 - **Risk:** Serverless workloads (Lambdas/Fargate) can quickly exhaust the database connection pool during traffic spikes.
 - **Mitigation:** We will deploy **AWS RDS Proxy** immediately in front of the Aurora cluster to handle intelligent connection pooling and multiplexing.
