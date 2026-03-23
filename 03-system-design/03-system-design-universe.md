# Alborz Bank — High-Level System Architecture


## The Alborz Universe: Cross-Team Communication
This diagram maps how the Customer interacts with the diverse engineering teams, and how those team s communicate with each other securely in the backend.

```mermaid
flowchart TD
    %% Define Beautiful Color Classes
    classDef customer fill:#f9e1e1,stroke:#d62828,stroke-width:3px,color:#000
    classDef auth fill:#e2ece9,stroke:#2a9d8f,stroke-width:2px,color:#000
    classDef onboarding fill:#fff3b0,stroke:#e9c46a,stroke-width:2px,color:#000
    classDef deposits fill:#ade8f4,stroke:#0077b6,stroke-width:2px,color:#000
    classDef payments fill:#d8b4e2,stroke:#9c27b0,stroke-width:2px,color:#000

    %% The 5 Worlds
    C((👤 Customer)):::customer
    T4[🔐 Team 4: Platform<br/>Identity & Auditing]:::auth
    T1[📋 Team 1: Onboarding<br/>KYC & Underwriting]:::onboarding
    T2[🏦 Team 2: Deposits<br/>Core Ledger]:::deposits
    T3[💸 Team 3: Payments<br/>Fiat Clearing]:::payments

    %% Customer Operations
    C -- "1. Logs in (BankID)" --> T4
    C -- "2. Submits Application" --> T1
    C -- "3. Views Balance" --> T2
    C -- "4. Requests Wire Transfer" --> T3
    C -- "5. Fetches Legal PDFs" --> T1

    %% Cross-Team Internal Communication
    T4 -. "Issues Auth-Token" .-> T1
    T4 -. "Issues Auth-Token" .-> T2
    T4 -. "Issues Auth-Token" .-> T3
    
    T1 -- "Publishes CustomerVerified Event" --> T2
    T2 -- "Publishes AccountMatured Event" --> T3
    T2 -- "Publishes PayoutRequested Event" --> T3
    T3 -- "Records Inbound Wire (Internal API)" --> T2
    
    T2 -- "Publishes Domain Event (PubSub)" --> T4
    T4 -- "Sends SMS/Email Notifications" --> C

    %% Telemetry
    T1 -. "Streams Logs/Audit" .-> T4
    T2 -. "Streams Logs/Audit" .-> T4
    T3 -. "Streams Logs/Audit" .-> T4
    
    %% Regulatory Reporting
    FinAuth((🏛️ Financial Authority))
    T3 -- "Transmits M30 Reports" --> FinAuth

```


## The Alborz Universe: End-to-End Data Flow
This sequence diagram illustrates a complete, high-level end-to-end customer journey bridging all 5 distinct worlds (The Customer and the 4 Engineering Teams) to visualize exactly how the microservices interlock to deliver the banking experience.

```mermaid
sequenceDiagram
    autonumber
    
    actor Cust as 👤 Customer
    participant T4 as 🔐 Team 4: Platform
    participant T1 as 📋 Team 1: Onboarding
    participant T2 as 🏦 Team 2: Deposits
    participant T3 as 💸 Team 3: Payments

    %% 1. Authentication & Onboarding
    Note over Cust, T3: Phase 1: Authentication & Onboarding
    Cust->>T4: Login with Swedish BankID
    T4-->>Cust: Issues Secure OAuth V2 Token
    
    Cust->>T1: Fetches Legal PDFs (Legal Document API)
    T1-->>Cust: Serves T&C/Privacy from S3
    
    Cust->>T1: Submits KYC & Initial Application
    T1->>T1: Underwriting Engine (Async checks)
    T1-->>Cust: Application Approved / Active
    
    %% 2. Core Ledger & Funding
    Note over Cust, T3: Phase 2: Core Ledger Generation & Funding
    T1-)+T2: Publishes "CustomerVerified" Event (Pub/Sub)
    T2->>T2: Listens & Opens Core Bank Account
    
    Cust->>T3: Wires Fiat Money from External Bank to IBAN
    T3->>T2: Parses inbound CAMT XML & Posts Deposit (PYI)
    T2->>T2: Appends Ledger Event (Increases Balance)
    
    %% 3. Maturity & Yield
    Note over Cust, T3: Phase 3: Interest Accrual & Maturity
    T2-)+T3: Publishes "AccountMatured" Event (Pub/Sub EventBus)
    T3->>T3: Async Tax Engine Calculates Regulatory M30 Yield Tax
    T3->>T2: Posts Tax Deduction (Internal Sync API)
    T2-)+T4: Publishes "AccountMatured" Event
    T4-)Cust: Notification Engine Sends "Account Matured" Email/SMS
    
    %% 4. Payout
    Note over Cust, T3: Phase 4: Final Payout & Clearing
    Cust->>T2: Requests Account Breaking / Withdrawal (CAN)
    T2->>T2: Calculates Penalties & Deducts Ledger
    T2->>T3: Publishes "PayoutRequested" Event (Pub/Sub EventBus)
    T3->>T3: Orchestrator Locks Job & Dispatches SEPA Wire
    T3-->>Cust: Fiat Transferred to External Bank
    
    %% Global Observability
    T1-->>T4: Streams Compliance Audit Logs (Firehose)
    T2-->>T4: Streams Ledger Telemetry (Firehose)
    T3-->>T4: Streams Payout Failures (Firehose)
    
    %% Regulatory Reporting
    Note over T3: End-of-Month Regulatory Trace
    T3->>T3: Regulatory Reporter CRON
    T3->>T3: Archives XML to S3 & Transmits to Financial Authority
```