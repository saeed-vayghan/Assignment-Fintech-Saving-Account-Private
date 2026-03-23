# Deposits Team: Architecture Explanation & Nuances

This document explains the architectural decisions behind dividing the Deposits team's domain into the specific microservices exactly as defined in the `02-deposits.md` profile.

### The 6 Microservices:
1. `Core Ledger Service`
2. `Transaction Processing Service`
3. `Notification Service`
4. `Account Maturity Cron Job`
5. `Ledger Projection Service`
6. `Customer Web BFF`

---

## 1. Are they Serverless?
**Yes, but with strict database coupling.** As defined in the `04-teams.md` document, the compute layer is **AWS Lambda**, but the database layer relies heavily on **Amazon Aurora (PostgreSQL) Serverless**. 

Unlike the Onboarding team which uses stateless NoSQL, the Deposits team must maintain persistent connection pools to a relational database to handle complex ACID transactions. AWS Lambda's concurrency model requires careful management (like RDS Proxy) when talking to relational databases to avoid exhausting connections during traffic spikes.

---

## 2. Why exactly these 4 boundaries? (The Challenge)

Let's challenge if we have too many or too few services by evaluating their Domain-Driven Design (DDD) boundaries and failure domains.

### 1. Core Ledger Service
* **Why it exists:** This service is the absolute source of truth for all balances. It handles the highly synchronous, customer-facing REST APIs for the Customer Web App (Dashboard) and processing withdrawal requests.
* **Challenge:** *Could we merge this with the Transaction Processing service?* No. The Core Ledger Service must be ultra-fast and directly available to end-users checking their balances via their mobile phones. If we bundled heavy, asynchronous internal batch-processing logic (like processing 50,000 monthly tax entries) into the same compute boundaries, an internal batch job could consume all database connections, causing the public-facing customer dashboard to timeout. The Core Ledger compute bounds must be isolated and prioritized for live users.

### 2. Transaction Processing Service
* **Why it exists:** This service acts as the internal Write-API and Orchestrator. It receives `PYI` (Pay-In), `IBR` (Interest), and `TAX` payloads from the Payments team, enforces idempotency, and securely writes them to the ledger. It also manages the complex multi-step workflow of customer-initiated account closures.
* **Challenge:** *Why not let the Payments team write directly to the Core Ledger API?* Separation of concerns and security. The Payments team does the math, but the Deposits team owns the money. By forcing Payments to talk to the Deposits `Transaction Processing Service`, the Deposits team can enforce strict validation rules, check account locking statuses, and apply back-dated logic before actually committing the transaction to Aurora. It acts as the gatekeeper.

### 3. Notification Service
* **Why it exists:** This service listens to `EventBridge` events (like `DepositReceived` or `MaturityReached`) and orchestrates calling AWS SES or SNS to text/email the user.
* **Challenge:** *Why separate it? Why not just send the email right after the Core Ledger writes the transaction to the database?* **Failure blast radius.** If AWS SES experiences an outage, or a user's email inbox bounces the message, that failure should absolutely never rollback or crash the Core Ledger database transaction. Sending emails is inherently asynchronous and flaky; committing financial math is highly synchronous and rigid.

### 4. Account Maturity Cron Job
* **Why it exists:** An EventBridge scheduled Lambda that wakes up at midnight, queries the Aurora database for all fixed-term deposits where the `maturity_date` equals today, and transitions their internal state to `MATURED`.
* **Challenge:** *Why not let Payments handle this since they run End-of-Month Cron jobs?* **Domain Data Ownership.** State transitions belong explicitly to the bounded context that owns the database. The Payments team does not own the `Deposits Ledger` Aurora database. Allowing a Payments CRON job to blindly write `UPDATE deposits SET state = 'MATURED'` would completely violate the strict API isolation of the Deposits team. Deposits must independently transition its own state and publish an `AccountMatured` EventBridge event for Payments to optionally listen to.

### 5. Ledger Projection Service
* **Why it exists:** To enable **CQRS (C10)**. It asynchronously consumes ledger events to build highly-optimized, lightning-fast "Read Models" for the customer dashboard.
* **Challenge:** *Why use a separate service? Why not just query the ledger directly?* **Performance and Isolation.** Complex SQL queries that calculate balances across millions of rows would slow down the core transaction write-path. By projecting events into a separate read-table, the dashboard query takes 10ms instead of 500ms, and it never locks the records that the Core Ledger is trying to write to.

### 6. Customer Web BFF
* **Why it exists:** To act as the specific **Backend for Frontend (C11)**. It simplifies the API experience for the mobile/web app by aggregating data from local Deposits services and the external Onboarding service into a single JSON response.
* **Challenge:** *Why not just let the Web App call 5 different microservices?* **User Experience (UX).** Making 5 separate API calls from a mobile device over a flaky 4G connection is slow and battery-intensive. The BFF handles that "chattiness" on the fast internal AWS network, sending only the final, perfectly formatted data to the user's phone in one single trip.

---

## Conclusion
The Deposits 6-microservice split protects the customer experience. Live users are shielded from batch job slowdowns, dashboards are decoupled from write-heavy databases via CQRS, and the web app is optimized via a dedicated BFF. By maintaining strict service boundaries, the architecture ensures financial integrity while delivering a premium, low-latency user interface.
