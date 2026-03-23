# Alborz Bank — Engineering Teams Architecture Entities

The following tables map every major technical entity owned by the four engineering teams, categorizing microservices, databases, messaging layers, and external integrations to provide a high-level system design overview.

## Team 1: Onboarding & Compliance
Handles customer registration, regulatory KYC/AML checks, and the automated underwriting pipeline.

| Category | Entity Name | Type / Comm. Model | Purpose |
| :--- | :--- | :--- | :--- |
| **Service** | Onboarding BFF | Sync | Acts as the dedicated frontend orchestration layer, aggregating T&C PDFs and initiating registration. |
| **Service** | Registration API | Sync | The core domain service that validates age/residency and handles internal DRAFT state transitions. |
| **Service** | IDV Webhook Handler | Async | Parses asynchronous identity verification callbacks. |
| **Service** | Underwriting Engine | Async (Step Function) | Evaluates final KYC/PEP rules to auto-approve or reject applications. |
| **Service** | Legal Document API | Sync | Serves region-specific, versioned T&C / Privacy PDFs prior to user signing. |
| **Database** | DynamoDB (Onboarding) | NoSQL Key-Value | High-speed, scalable state tracking for applications and raw webhooks. |
| **Database** | Amazon S3 (Legal Assets) | Object/Blob Storage | Stores immutable versions of all explicit legal and fee agreements. |
| **External Service** | Third-Party IDV (Signicat/Onfido) | Async Webhook | Parses passports, selfies, and proof-of-address documents. |
| **External Service** | Peps/Sanctions (ComplyAdvantage) | Sync API | Screens applicants against global watchlists in real-time. |
| **Messaging/Queue** | EventBridge (Central Bus) | Pub/Sub Event Bus | Publishes `KYC_SUBMITTED` events to trigger parallel domain operations. |

---

## Team 2: Deposits & Transactions
The core financial ledger. Strictly enforces ACID mathematical consistency and account life-cycles.

| Category | Entity Name | Type / Comm. Model | Purpose |
| :--- | :--- | :--- | :--- |
| **Service** | Customer Web BFF | Sync | The dedicated frontend API for the web app to safely query cached balances and transaction history. |
| **Service** | Core Ledger API | Sync | The isolated core domain API that strictly processes ACID PYI/PYO financial commands. |
| **Service** | Transaction Processing Service | Async (Scheduled) | Automatically calculates daily account yield and matures `FIXED_TERM` deposits at midnight. |
| **Service** | Projection Builder | Async (CDC Stream) | Reads PostgreSQL WAL to asynchronously populate the CQRS read model. |
| **Database** | Aurora PostgreSQL | Relational (RDBMS) | The source of truth for all financial mathematically strict ledger events. |
| **External Service** | None | N/A | Fully encapsulated core domain (Zero external dependencies). |
| **Messaging/Queue** | EventBridge (Central Bus) | Pub/Sub Event Bus | Publishes `AccountMatured` and `PayoutRequested` domain events. |
| **Caching Layer** | Materialized CQRS View | DB-Level Read Cache | The `Customer_Balance` view acts as a high-speed cache for the UI. |

---

## Team 3: Payments & Accounting
Translates external banking files (CAMT) into internal API calls, and processes massive outgoing fiat payouts.

| Category | Entity Name | Type / Comm. Model | Purpose |
| :--- | :--- | :--- | :--- |
| **Service** | Payout Orchestrator | Async | Consumes `PayoutRequested` events and negotiates with external clearing banks. |
| **Service** | Inbound CAMT Parser | Async | Ingests raw XML payout batches and forwards them to the Deposits API. |
| **Service** | Tax Withholding Engine | Async | Calculates end-of-month (M30) regulatory government taxes on generated interest. |
| **Service** | Regulatory Reporter | Async (Scheduled) | Generates and dispatches mandatory end-of-month financial reports to the central banking authority. |
| **Database** | DynamoDB (Payments) | NoSQL Key-Value | Maintains strict idempotency and job-locking to prevent "Double Payouts". |
| **Database** | Amazon S3 (Regulatory Archive) | Object/Blob Storage | Auditable 10-year storage of all generated central bank reports. |
| **Database** | Amazon S3 (Settlement Archive) | Object/Blob Storage | Archives raw CAMT053 XML files immutably for 7-10 years. |
| **External Service** | Partner Clearing Bank (SEPA) | SFTP / Async API | Physically moves fiat currency between the central bank and external banks. |
| **Messaging/Queue** | EventBridge (Central Bus) | Pub/Sub Event Bus | Subscribed to listen for withdrawals and account maturity jobs from Team 2. |

---

## Team 4: Platform & Shared
Cross-cutting infrastructure supporting global authentication, observability, and network routing.

| Category | Entity Name | Type / Comm. Model | Purpose |
| :--- | :--- | :--- | :--- |
| **Service** | Auth API | Sync | The core identity service that handles Authentications like BankID, Creds, etc. |
| **Service** | Refresh Token Service | Sync | Manages active sessions and allows instantaneous global revocation. |
| **Service** | Log Collector | Async | Aggregates massive API Gateway logs into OpenSearch without blocking requests. |
| **Service** | Notification Engine | Async (Event-Driven) | Listens to EventBridge to asynchronously fire emails and SMS to customers. |
| **Database** | DynamoDB (Auth) | NoSQL Key-Value | Ultra-low latency storage for Auth Tokens. |
| **Database** | OpenSearch / ELK | Document / Search | The centralized search index for company-wide Telemetry and Compliance Auditing. |
| **External Service** | BankID / Freja eID | Sync OIDC API | Master federated identity provider for Swedish citizens. |
| **External Service** | AWS SES / SNS | Async API | Managed infrastructure for globally delivering transactional banking emails and texts. |
| **Messaging/Queue** | Kinesis Firehose | Real-Time Stream | Buffers and reliably routes millions of JSON logs into OpenSearch index rollovers. |
