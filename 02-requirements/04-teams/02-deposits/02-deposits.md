# Team 2: Deposits & Transactions

### **Purpose**:
Manage the core banking ledger (M5, M6), including fixed term deposits, overnight savings, transaction processing histories (M7), customer notifications (M33), and the customer web application interface (M12).

### **UI Applications**:
1. **Customer Web App:** A React/Next.js portal for end-users to view account balances (M12). Powered by the **Customer Web BFF (C11)** which aggregates data from Onboarding and the Ledger.
2. **Back-office Admin Portal:** An internal ops tool (S4) to view customer accounts, resolve issues, and trigger manual payouts.
3. **Interest Rate Admin Tool:** A specific UI tool managing product configuration. Integrated via the **Back-office Operations BFF**.

### **User-Facing Service**:
Yes. The Customer Web App and the underlying transaction APIs are the primary interface for active, verified customers interacting with their savings.

### **Authentication / Authorization Mechanisms**:
*(See `06-auth-service.md` for Centralized Issuance / Decentralized AuthZ strategy)*
- **Customer Web App (Dashboard):** Relies on `eID` or `Local-Cred` (AuthN) brokered by the Platform team.
- **Core Ledger / Transactions API:** The API Gateway validates the `OAuth-Token`. Deposits logic relies on the injected `customerId` header for strict decentralized AuthZ (preventing cross-account reads).
- **Admin & Interest Rate Portals:** Requires strict `OAuth-RBAC` (`Role: Financial_Admin`) via Corporate `SSO`.

### **Microservices**:
  1. `Core Ledger Service`: Handles ACID-compliant deposit (M5, M6) and withdrawal logic (S1). Enforces strict idempotency on all write operations to prevent double-charging (M22).
  2. `Transaction Processing Service`: Manages the processing of `PYI, PYO, CAN, IBR, TAX` entries (M7) and full account closures (M37).
  3. `Notification Service`: Triggers transactional emails and SMS (M33) for deposit received, maturity, payout, rate change, and compliance.
  4. `Account Maturity Cron Job`: An EventBridge-scheduled Lambda that scans the ledger daily at midnight to transition fixed-term deposits from `ACTIVE` to `MATURED`.
  5. `Ledger Projection Service`: Asynchronously consumes ledger events to build highly-optimized CQRS read-models for the customer dashboard (C10).
  6. `Customer Web BFF`: A standalone serverless API layer aggregating account, transaction, and onboarding state for the React/Next.js frontend (C11).

### **Databases & Storage Access**:

| Database / Storage | Type | Access Level | Purpose / Justification |
|--------------------|------|--------------|-------------------------|
| **Amazon Aurora (PostgreSQL)** (Core Ledger) | Relational SQL (Serverless) | **Read / Write** (Owned) | **ACID Compliance:** Mandatory strict ledger logic to guarantee money is never lost or duplicated.<br>**Mathematical Rigor:** Exact data types to prevent floating-point errors.<br>**Relational Data:** Complex joins across Customers, Accounts, Rates, and Transactions. |
| **ELK Stack** (Central Logging) | Search Index | **Write** (via Platform API) | **Cross-Team Dependency:** All transactional audit logs and notification events are pushed to the central observability cluster owned by **Team 4 (Platform)**. |

### **Caching Layer**:
No caching for core balances (needs strong consistency to prevent overdrafts). Edge caching (CDN) is used for static web app assets.

### **Message Queue / Async Comm**:
- Uses `AWS EventBridge` to publish `DepositCreated`, `AccountMatured`, and `TransactionCommitted` events.
- Uses **`AWS EventBridge Scheduler`** to trigger the nightly Account Maturity CRON.
- Uses an **`AWS SQS (FIFO)` Bulk Transaction Buffer** to absorb massive end-of-month `IBR` (Interest) write-requests from the Payments team, purposefully rate-limiting processing to protect Aurora PostgreSQL from connection starvation.

### **External Systems**:
The end-user's browser/device accessing the Web App.

### **Third-Party Services**:
  - AWS SES (Sending transactional emails)
  - AWS SNS (Sending SMS alerts)

### **Other Team Interactions**:
  - **Platform Team**: Leverages their API Gateway, WAF, and Auth-Service.
  - **Onboarding Team**: Listens to Onboarding's `CustomerVerified` events to know when to initialize an empty ledger account for a freshly approved user.
  - **Payments Team**: Accepts synchronous API calls or asynchronous events from Payments to solidly book final `PYI` (Pay-In), `IBR` (Interest), and `TAX` transactions from batch processed files.

### **Edge Cases & Failure Scenarios**:

| Category | Scenario | Immediate Strategy / Impact | HITL Action / Threshold |
|----------|----------|-----------------------------|-------------------------|
| **Retryable** | **DB Lock / Concurrency:** Two simultaneous withdrawals on the same ledger. | DB transaction fails with serialization error, API auto-retries. | **Support (Moderate):** If >3 fails, returns 409 Conflict to user to try again later. |
| **Retryable** | **Notification Delivery Failure:** AWS SES drops the maturity email. | EventBridge/DLQ retries sending. | **Ops Team (Moderate):** Alerted to investigate SES bounce rates. |
| **Retryable** | **Transaction Buffer Spikes:** EOM interest pushes 100,000 pending SQS messages. | SQS buffers safely; Lambda consumers process at DB-safe concurrency limit. | **FinOps (Moderate):** If SQS age > 2 hours, manually scale up Aurora Writer capacity. |
| **Moderate** | **Negative Balance from Back-dated Fee:** A delayed withdrawal pushes account < 0. | Strict ACID DB constraints immediately halt the transaction. | **Financial Ops:** Manually reviews the ledger discrepancy to resolve fee logic. |
| **Moderate** | **Term Deposit Early Withdrawal Dispute:** Customer complains about penalty. | Standard logic applies the penalty. | **Customer Support:** Uses Admin Portal (S4) to manually override penalty if justified. |
| **Major** | **Ledger Imbalance / Double-Spend:** Bug allows withdrawal > current balance. | Core ledger goes out of sync with history. | **Financial Controllers:** Temporarily freeze account and manually reconcile ledger. |

### **Logs and Metrics**:
Strict financial audit logs, API latency metrics, and error rates for transaction processing. Tracks notification delivery rates (M33).

### **API Exposed**:
  - External REST APIs (fetched via the Customer Web App for getting balances and requesting withdrawals)
  - Internal REST APIs (for the Payments team to book verified entries)
  - Internal REST APIs (Back-office and Rate Management Portals)

### **Roadmap & MoSCoW Prioritization References**:
#### **Must Have (MVP)**
- `M5`: Fixed term deposit: create, mature, rollover
- `M6`: Overnight savings account: create, top-up, partial withdraw
- `M7`: Transaction processing: PYI, PYO, CAN, IBR, TAX
- `M12`: Customer web app: account dashboard & transaction history
- `M22`: Idempotency on all write operations
- `M33`: Customer notifications: email/SMS for deposit received, maturity, payout, rate change, compliance status
- `M35`: Interest rate management: admin tool for setting/changing rates, effective date logic, rate change history
- `M37`: Account closure flow: customer-initiated full account closure

#### **Should Have**
- `S1`: Customer web app: withdrawal request flow
- `S4`: Back-office admin portal: view customer accounts, resolve issues, trigger manual payouts
