# Alborz Bank — High-Level System Architecture of All Teams

---

## Team 1: Onboarding & Compliance

```mermaid
flowchart LR
    classDef api fill:#fff3b0,stroke:#e9c46a,stroke-width:2px,color:#000
    classDef bff fill:#ade8f4,stroke:#0077b6,stroke-width:2px,color:#000

    UI(Web App) --> BFF(Onboarding BFF):::bff
    BFF --> LegalAPI(Legal Document API):::api
    BFF --> API(Registration API):::api
    LegalAPI --> LegalS3[(Amazon S3:<br/>Legal Assets)]:::db
    API --> DB[(DynamoDB:<br/>Applications)]:::db
    
    IDV_Vend((IDV Providers<br/>Signicat/Onfido)):::ext -- "Webhook payload" --> WH(Webhook Handler):::api
    WH --> DB
    
    AML_Vend((AML/PEP<br/>ComplyAdvantage)):::ext <-- "Checks Watchlists" --> UWE{Underwriting<br/>Step Function}:::api
    
    DB -- "Triggers Engine" --> UWE
    UWE -- "Updates Decision" --> DB
```

---

## Team 2: Deposits & Transactions

```mermaid
flowchart TD
    classDef api fill:#ade8f4,stroke:#0077b6,stroke-width:2px,color:#000
    classDef db fill:#caf0f8,stroke:#03045e,stroke-width:2px,color:#000
    classDef cron fill:#90e0ef,stroke:#00b4d8,stroke-width:2px,color:#000
    classDef bff fill:#ade8f4,stroke:#0077b6,stroke-width:2px,color:#000

    Internal(Onboarding / Payments) --> API(Core Ledger API):::api
    UI(Web App) --> BFF(Deposits BFF):::bff
    BFF -- "Reads Cache" --> RL
    
    API -- "Appends Mutation" --> WL[(Aurora Postgres:<br/>Ledger_Events)]:::db
    
    WL -- "CDC Stream (WAL)" --> Proj(Projection Builder):::api
    Proj -- "Calculates Balance" --> RL[(Aurora Postgres:<br/>Customer_Balance_View)]:::db
    
    UI(Web App) -- "Reads Cache" --> RL
    
    Cron((Midnight Cron)):::cron -- "Triggers" --> Mat(Transaction Processing Engine):::api
    Mat -- "Calculates Yield & Matures" --> WL
```

---

## Team 3: Payments & Accounting

```mermaid
flowchart LR
    classDef api fill:#d8b4e2,stroke:#9c27b0,stroke-width:2px,color:#000
    classDef db fill:#e9c46a,stroke:#e76f51,stroke-width:2px,color:#000
    classDef ext fill:#ece4db,stroke:#8a817c,stroke-width:2px,color:#000,stroke-dasharray: 5 5
    classDef cron fill:#90e0ef,stroke:#00b4d8,stroke-width:2px,color:#000

    EventBus((EventBus<br/>PayoutRequested)) --> Orch(Payout Orchestrator):::api
    Orch -- "Locks Job" --> DB1[(DynamoDB:<br/>Payout Jobs)]:::db
    Orch -- "Dispatches API" --> SEPA((Central Bank<br/>SEPA/SWIFT)):::ext
    
    SEPA -- "SFTP Upload" --> S3[(Amazon S3:<br/>CAMT Archive)]:::db
    S3 -- "S3 Trigger" --> Parser(Inbound CAMT Parser):::api
    Parser -- "Idempotency Lock" --> DB2[(DynamoDB:<br/>Inbound Locks)]:::db
    Parser -- "Calls Deposits API" --> Core(Team 2:<br/>Deposits Ledger)
    
    EventBus2((EventBus<br/>AccountMatured)) --> Tax(Tax Engine):::api
    Tax -- "Calculates Due" --> DB3[(DynamoDB:<br/>Tax Ledger)]:::db
    
    Cron((End of Month<br/>Cron)):::cron -- "Triggers" --> RegRep(Regulatory Reporter):::api
    RegRep -- "Uploads XML" --> RegS3[(Amazon S3:<br/>Regulatory Archive)]:::db
    RegRep -- "Transmits" --> FinAuth((Financial Authority)):::ext
```

---

## Team 4: Platform & Shared

```mermaid
flowchart TD
    classDef api fill:#e2ece9,stroke:#2a9d8f,stroke-width:2px,color:#000
    classDef db fill:#c0efd1,stroke:#264653,stroke-width:2px,color:#000
    classDef ext fill:#ece4db,stroke:#8a817c,stroke-width:2px,color:#000,stroke-dasharray: 5 5

    UI(Web App) --> API(Auth API):::api
    BankID((Swedish BankID)):::ext <-- "OIDC Flow" --> API
    
    API -- "Issues / Validates" --> AuthDB[(DynamoDB:<br/>Tokens & Users)]:::db
    
    AllTeams((All Microservices)) -- "Emits Logs to API Gateway" --> Kinesis(Kinesis Firehose):::api
    Kinesis -- "Batches & Masks PII" --> ELK[(OpenSearch:<br/>Logs & Audit)]:::db
    
    SOC(SecOps / Compliance) -- "Searches" --> ELK
    
    EventBus((EventBus<br/>PubSub Events)) --> Notify(Notification Engine):::api
    Notify -- "Sends SMS/Email" --> SES((AWS SES/SNS)):::ext
```

---
