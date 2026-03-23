
### Preface Summary:
- savings account for individuals
- Individuals can open savings accounts in Alborz Bank and deposit money
- fixed term deposits: 3, 6 or 12 months with fixed interest rate

### Business information Summary:
- user onboarding
- user identification
- KYC information gathering

---

### Business Requirements Summary:

#### 1. Customer Eligibility

- **Who can open an account:**
    - Individual (private) persons only.
    - Must be a resident of the same country where the Alborz Bank branch operates.
- **Who is NOT eligible:**
    - Minors (under 18).
    - Trusts, businesses, governmental entities, or any non-individual entity.

#### 2. Deposit Types

- **Fixed Term Deposit:**
    - The customer locks in a specific amount of money for a set period: **3 months, 6 months, or 12 months**.
    - The interest rate is **fixed** — it does not change during the term.
    - At the end of the term (maturity), the deposit can be **rolled over** (automatically renewed for another term).
    - No partial withdrawals are allowed during the term. Early cancellation follows the `CAN` transaction flow (see below).

- **Overnight Savings Account:**
    - A flexible savings account with no lock-in period.
    - The interest rate is **variable** — it can change over time.
    - Interest is **calculated daily** based on the account balance.
    - The customer can **top up** (add more money) or **partially withdraw** funds at any time.

#### 3. Transaction Types

Each money movement or accounting event in the system is recorded as one of the following transaction types:

| Code  | Full Name              | Description                                                                                                    |
|-------|------------------------|----------------------------------------------------------------------------------------------------------------|
| `PYI` | **Pay-In**             | Money received from the customer's external bank account into their deposit account at Alborz Bank. This covers the initial deposit and any later top-ups (overnight accounts only). |
| `PYO` | **Pay-Out**            | Money sent back from the Alborz Bank deposit account to the customer's verified external bank account. This happens when a fixed term deposit matures and is not renewed, or when a partial withdrawal is made from an overnight account. |
| `CAN` | **Cancellation**       | Early (irregular) termination of a fixed term deposit before the maturity date. The full deposit amount is returned to the customer's external bank account. May involve penalty or reduced interest. |
| `IBR` | **Interest Booking**   | An internal accounting entry that records interest earned on the deposit. **No real money moves** — this is a ledger-only transaction that tracks accrued interest. |
| `TAX` | **Tax Deduction**      | Withholding tax deducted at the source. This reduces the final amount the customer receives at payout. Recorded as a separate transaction for audit and reporting purposes. |

#### 4. Payment Import (Inbound Payments)

- Incoming payments arrive at a bank account denominated in the **market default currency**.
- These payments are exported as **CAMT files** (ISO 20022 standard XML format for bank-to-customer statements).
- The CAMT files are placed on an **SFTP server** for the system to pick up.
- The system must:
    1. Parse each CAMT file.
    2. Match each payment to the correct customer account using the **payment reference**.
    3. Create the corresponding `PYI` transaction in the system.

#### 5. Accounting Jobs (Background Processes)

The system must run scheduled background jobs to handle the following:

- **Payout processing:**
    - For matured or non-renewed fixed term deposits → create a `PYO` transaction and send money back to the customer.
    - For early cancellations → create a `CAN` transaction and send money back to the customer.
    - For overnight account withdrawal requests → create a `PYO` transaction for the requested amount.

- **Interest calculation:**
    - Calculate interest based on the deposit type and applicable rate.
    - Record the result as an `IBR` transaction (ledger entry only, no real money movement).

- **Tax calculation:**
    - Calculate withholding tax on accrued interest.
    - Record the result as a `TAX` transaction (reduces the eventual payout amount).

---

### Legal & Compliance Requirements

All four steps below are **mandatory** and must be completed before the account can be activated. The recommended processing order is: **Identity Verification → Bank Account Validation → PEP/Sanctions Screening → KYC Questionnaire**.

#### Step 1: Identity Verification (Blocking)

The system must collect and verify the customer's identity before proceeding with account creation.

- **Required data from the customer:**
    - Full legal name
    - Date of birth
    - Nationality
    - National ID number or passport number
    - Proof of address (e.g. utility bill, bank statement)

- **Verification process:**
    - Validate the format and consistency of the submitted documents.
    - Optionally call a **third-party identity verification provider** (e.g. Onfido, IDnow) to perform automated document and biometric checks.
    - Store the verification result and status (`VERIFIED`, `PENDING`, `REJECTED`) against the customer record.

- **Blocking:** The customer cannot proceed to the next step until identity verification is completed successfully.

#### Step 2: Bank Account Validation (Blocking)

The system must validate the customer's external bank account. This account is used for all future pay-ins (`PYI`) and pay-outs (`PYO` / `CAN`).

- **Validation checks:**
    - **IBAN format validation** — Verify the IBAN structure and check digits are correct.
    - **Ownership verification** — Confirm that the bank account belongs to the customer. This can be done via:
        - A **micro-deposit** (send a small amount, ask customer to confirm it), or
        - A **name-matching API** call to an external banking data provider, or
        - A **bank account verification service** (e.g. Plaid, Tink).

- **Result:** The bank account is marked as `VERIFIED` and stored as the customer's **verified transaction account**. All future payouts will only go to this account.

- **Blocking:** The account cannot be activated until the bank account is verified.

#### Step 3: PEP/Sanctions Screening (Non-Blocking, but Requires Resolution)

The system must check whether the customer appears on any PEP (Politically Exposed Person) or international Sanctions lists.

- **Process:**
    - Send the customer's name, date of birth, and nationality to a **screening provider** (e.g. ComplyAdvantage, Refinitiv World-Check).
    - The provider returns a match result: `NO_MATCH`, `POTENTIAL_MATCH`, or `CONFIRMED_MATCH`.

- **If a match is found:**
    - The customer's record is flagged with a `WARNING` status.
    - A **compliance case** is created and assigned to a compliance officer for manual review.
    - The compliance officer can either **approve** (allow account creation) or **reject** (block the customer).

- **Non-blocking with condition:** The screening runs automatically, but if a match is found, account activation is paused until a compliance officer resolves the case.

#### Step 4: KYC Questionnaire (Blocking)

The customer must complete a KYC (Know Your Customer) questionnaire as part of the onboarding process.

- **Questionnaire structure:**
    - Consists of **5 questions** (configurable by the compliance/product team).
    - Each question can be one of the following types:
        - **Single-select** — Pick one option from a predefined list.
        - **Multi-select** — Pick one or more options from a predefined list.
        - **Free text** — Open-ended text input.
    - Questions and answer options should be **admin-configurable** (stored in a database or config, not hardcoded).

- **Storage:** All answers are stored against the customer record for **audit and regulatory reporting** purposes.

- **Blocking:** The questionnaire must be fully completed before the account can be activated.


## Technical assignment
Design a flow where you implement a new product - fixed term deposit & overnight savings accounts. In your scope you need to consider the onboarding process, underwriting validations as well as internal jobs and processes.
In your technical assignment you are expected to provide:

- Architecture (sequence & process) diagrams of the following flows:

    - **Onboarding Flow:**
        - Show the full customer journey from registration to account activation.
        - Include all 4 compliance steps (Identity Verification → Bank Account Validation → PEP/Sanctions Screening → KYC Questionnaire) and how they interact with external services.
        - Show decision points: what happens when a step fails, when PEP screening returns a match, or when the customer abandons mid-flow.
        - Include API calls between frontend, backend services, and third-party providers.
        - Show how customer state transitions (e.g. `DRAFT` → `PENDING_VERIFICATION` → `PENDING_SCREENING` → `ACTIVE` or `REJECTED`).

    - **Customer Experience — Web Application:**
        - Show the flow for a logged-in customer viewing their deposit accounts (both fixed term and overnight).
        - Include the flow for making a **withdrawal request** (partial withdrawal for overnight, early cancellation for fixed term).
        - Include the flow for viewing **transaction history** (PYI, PYO, CAN, IBR, TAX entries).
        - Show how the frontend communicates with backend APIs (REST or GraphQL), including authentication and authorization.
        - Cover how real-time or near-real-time account balance and interest data is presented to the customer.

- Considerations around best practices:

    - **Security:**
        - Authentication & authorization strategy (Auth-Service).
        - Data encryption: at rest (e.g. AWS KMS) and in transit (TLS).
        - Secrets management (e.g. AWS Secrets Manager, SSM Parameter Store).
        - API security: rate limiting, input validation, CORS policies.
        - PII (Personally Identifiable Information) handling and data masking in logs.
        - Role-based access control (RBAC) for internal tools (e.g. compliance officer dashboard).

    - **Monitoring:**
        - Application-level metrics: request latency, error rates, throughput.
        - Business-level metrics: number of accounts opened, deposits processed, screening flags raised.
        - Infrastructure monitoring: CPU, memory, Lambda invocations, API Gateway 4xx/5xx rates.
        - Alerting strategy: what triggers an alert, severity levels, escalation paths.
        - Tools: AWS CloudWatch, CloudTrail, AWS Config, Grafana dashboards.

    - **Logging:**
        - Structured logging format (JSON) with correlation IDs for tracing requests across services.
        - Log levels: what gets logged at INFO, WARN, ERROR.
        - Centralized log aggregation (e.g. CloudWatch Logs, ELK Stack).
        - Audit logging: all compliance-related actions (screening results, KYC submissions, account status changes) must be logged immutably for regulatory audit.
        - PII redaction in logs — sensitive fields like national ID, IBAN must be masked.

    - **Operational Excellence:**
        - Runbooks for common operational tasks (e.g. "how to manually retry a failed CAMT import", "how to re-run interest calculation for a specific date").
        - Incident response playbooks: define severity levels (P1–P4), escalation paths, and on-call rotation.
        - Post-incident review process: blameless retrospectives to capture lessons learned and prevent recurrence.
        - Operational readiness reviews (ORR) before each major release — a checklist ensuring monitoring, alerting, rollback plans, and documentation are in place.
        - Change management: all production changes go through a review and approval process.

        - **Developer Experience (sub-category):**
            - Infrastructure as Code using **AWS CDK** or CloudFormation for reproducible environments.
            - CI/CD pipeline design: build, test, deploy stages with automated quality gates.
            - Local development setup: how engineers can run and test services locally (e.g. LocalStack, Docker Compose).
            - Environment strategy: development, staging, production — with clear promotion process.
            - API documentation strategy (e.g. OpenAPI/Swagger auto-generated from code).
            - Feature flags for gradual rollout of new features. (experimentation platform)

    - **Reliability:**
        - Multi-AZ deployment for all critical services (databases, queues, compute) to survive availability zone failures.
        - Retry policies with exponential backoff for all external service calls (identity verification, PEP screening, bank validation APIs).
        - Dead-letter queues (DLQ) for failed async events — e.g. CAMT file processing failures, failed payout attempts — so no transaction is silently lost.
        - Idempotency on all write operations: processing the same CAMT entry or payout request twice must not create duplicate transactions.
        - Circuit breaker pattern for third-party integrations: if a provider is down, degrade gracefully instead of cascading failures.
        - Automated database backups with point-in-time recovery (e.g. AWS RDS automated backups).
        - Disaster recovery plan: define RPO (Recovery Point Objective) and RTO (Recovery Time Objective) targets for the system.
        - Health checks and auto-healing: unhealthy instances are automatically replaced (e.g. ECS task restart, Lambda retry).

    - **Performance Efficiency:**
        - Choose the right compute model for each workload:
            - **AWS Lambda** for event-driven, short-lived tasks (e.g. CAMT file parsing, webhook handlers).
            - **ECS/Fargate** for long-running or high-throughput services (e.g. interest calculation batch jobs).
        - Caching strategy: use caching (e.g. ElastiCache / Redis) for frequently read data such as interest rates, account summaries, and KYC question configurations.
        - Database optimization: proper indexing, read replicas for query-heavy workloads (e.g. transaction history), and connection pooling.
        - Async processing for non-blocking operations: payout execution, interest/tax calculation, and PEP screening should run asynchronously via queues (e.g. SQS, EventBridge).
        - Auto-scaling policies: define scaling triggers based on request volume, queue depth, or CPU utilization.
        - API response time targets: define SLAs (e.g. < 200ms for account balance queries, < 2s for onboarding step submissions).
        - Pagination for large datasets: transaction history and account listings must support cursor-based or offset pagination to avoid loading entire result sets.

    - **Cost Optimization:**
        - Use serverless compute (Lambda, Fargate) to avoid paying for idle resources — pay only for what you use.
        - Right-size database instances based on actual workload; consider **Aurora Serverless** for variable or unpredictable traffic patterns.
        - S3 lifecycle policies for CAMT file storage: move processed files to **S3 Infrequent Access** after 30 days, then to **S3 Glacier** after 90 days for long-term regulatory retention.
        - Use **Reserved Instances** or **Savings Plans** for predictable baseline workloads (e.g. always-on API services).
        - Apply **cost allocation tags** on all AWS resources (e.g. `service`, `team`, `environment`) to track and attribute spend.
        - Set up **AWS Budgets** and **CloudWatch billing alarms** to detect unexpected cost spikes early.
        - Regularly review unused or underutilized resources using **AWS Trusted Advisor** or **AWS Cost Explorer**.

    - **Sustainability:**
        - Minimize over-provisioning by using auto-scaling and serverless — run only what is needed, when it is needed.
        - Use **AWS managed services** (RDS, SQS, ElastiCache) over self-managed infrastructure to benefit from AWS's optimized, shared resource pools.
        - Schedule batch jobs (interest calculation, tax calculation) during **off-peak hours** to take advantage of lower grid carbon intensity periods where possible.
        - Set data retention policies: archive or delete data that is no longer needed for business or regulatory purposes, reducing storage footprint.
        - Use **AWS Graviton** (ARM-based) instances where supported — they deliver better performance per watt compared to x86 instances.

- Changes to existing systems to go live within one year:

    - Identify which existing systems need to be modified or extended (e.g. customer database, payment gateway integrations, notification services).
    - Define new services that need to be built from scratch (e.g. deposit management service, interest calculation engine, CAMT file processor).
    - Data migration or schema changes required in existing databases.
    - Integration points with existing internal services (e.g. authentication, customer profiles, notification systems).
    - Phased rollout plan: what can be delivered in MVP vs. full launch (e.g. start with fixed term deposits only, add overnight accounts later).
    - Risk assessment: key technical risks and mitigation strategies.
    - Team capacity and skill requirements: what roles are needed, any upskilling required.


## Nice to have:

- **Database entity-relationship diagram:**
    - Cover core entities: Customer, Account, Deposit, Transaction, KYC Response, Compliance Case, etc.
    - Show relationships: a Customer has many Accounts, an Account has many Transactions, etc.
    - Include key fields and data types for each entity.
    - Consider how to handle audit trails (e.g. separate audit tables or event sourcing).

- **Sample API documentation:**
    - Provide example endpoints for key operations (e.g. `POST /customers`, `GET /accounts/{id}`, `POST /withdrawals`).
    - Include request/response schemas with sample payloads.
    - Document authentication requirements, error codes, and pagination strategy.
    - Consider API versioning strategy (e.g. URL path versioning `/v1/...` or header-based).
    - Consideration on whether REST, GraphQL, or a combination is more appropriate for different use cases (e.g. REST for backend-to-backend, GraphQL for frontend flexibility).

- **Stakeholder questions:**
    - List open questions that would affect architectural decisions (e.g. expected transaction volume, multi-currency support, regulatory requirements in specific markets).
    - Include questions about existing system constraints, team preferences, and timeline expectations.

