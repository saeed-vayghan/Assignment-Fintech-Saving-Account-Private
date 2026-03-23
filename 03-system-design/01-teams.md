# Alborz Bank — Engineering Teams Architecture Entities

The following tables map every major technical entity owned by the four engineering teams, categorizing microservices, databases, messaging layers, and external integrations to provide a high-level system design overview.

## Team 1: Onboarding & Compliance
Handles customer registration, regulatory KYC/AML checks, and the automated underwriting pipeline.

| Category | Entity Name | Type / Comm. Model | Purpose |
| :--- | :--- | :--- | :--- |
| **Service** | Onboarding API | Sync | Handles initial DRAFT application creation and state transitions. |
| **Service** | IDV Webhook Handler | Async | Parses asynchronous identity verification callbacks. |
| **Service** | Underwriting Engine | Async (Step Function) | Evaluates final KYC/PEP rules to auto-approve or reject applications. |
| **Database** | DynamoDB (Onboarding) | NoSQL Key-Value | High-speed, scalable state tracking for applications and raw webhooks. |
| **External Service** | Third-Party IDV (Signicat/Onfido) | Async Webhook | Parses passports, selfies, and proof-of-address documents. |
| **External Service** | Peps/Sanctions (ComplyAdvantage) | Sync API | Screens applicants against global watchlists in real-time. |
| **Messaging/Queue** | EventBridge (Central Bus) | Pub/Sub Event Bus | Publishes `KYC_SUBMITTED` events to trigger parallel domain operations. |

---

## Team 2: Deposits & Transactions
The core financial ledger. Strictly enforces ACID mathematical consistency and account life-cycles.

| Category | Entity Name | Type / Comm. Model | Purpose |
| :--- | :--- | :--- | :--- |
| **Service** | Core Ledger API | Sync | Processes PYI/PYO commands and executes Postgres ACID transactions. |
| **Service** | Maturity Cron Job | Async (Scheduled) | Automatically rolls over or matures `FIXED_TERM` accounts at midnight. |
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
| **Database** | DynamoDB (Payments) | NoSQL Key-Value | Maintains strict idempotency and job-locking to prevent "Double Payouts". |
| **Database** | Amazon S3 (Settlement Archive) | Object/Blob Storage | Archives raw CAMT053 XML files immutably for 7-10 years. |
| **External Service** | Partner Clearing Bank (SEPA) | SFTP / Async API | Physically moves fiat currency between the central bank and external banks. |
| **Messaging/Queue** | EventBridge (Central Bus) | Pub/Sub Event Bus | Subscribed to listen for withdrawals and account maturity jobs from Team 2. |

---

## Team 4: Platform & Shared
Cross-cutting infrastructure supporting global authentication, observability, and network routing.

| Category | Entity Name | Type / Comm. Model | Purpose |
| :--- | :--- | :--- | :--- |
| **Service** | Auth API | Sync | Authenticates users against BankID and securely issues JWT tokens. |
| **Service** | Refresh Token Service | Sync | Manages active sessions and allows instantaneous global revocation. |
| **Service** | Log Collector | Async | Aggregates massive API Gateway logs into OpenSearch without blocking requests. |
| **Database** | DynamoDB (Auth) | NoSQL Key-Value | Ultra-low latency storage for JWT expiry validation and identity mapping. |
| **Database** | OpenSearch / ELK | Document / Search | The centralized search index for company-wide Telemetry and Compliance Auditing. |
| **External Service** | BankID / Freja eID | Sync OIDC API | Master federated identity provider for Swedish citizens. |
| **Messaging/Queue** | Kinesis Firehose | Real-Time Stream | Buffers and reliably routes millions of JSON logs into OpenSearch index rollovers. |
