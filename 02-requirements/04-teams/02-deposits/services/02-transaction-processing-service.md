# Transaction Processing Service (Maturity & Interest)

## What is it?
It is responsible for calculating daily interest yield, understanding when fixed-term deposits mature, and automatically compounding interest.

## Core Logic & Rules
1. **Time-Driven Execution:** Relies on strict nightly CRON triggers to evaluate the state of every account.
2. **Yield Calculation:** Determines the exact fraction of interest owed to the customer based on the specific `Product_Catalog` rates assigned to their account at opening.
3. **Maturity Triggers:** When a fixed-term account naturally expires (e.g., a 1-year lock ends), this service publishes the critical `AccountMatured` domain event so the Payments team can physically move the fiat.

## Data Flow Visualization
```mermaid
flowchart TD
    %% Colors
    classDef trigger fill:#e2ece9,stroke:#2a9d8f,stroke-width:2px,color:#000
    classDef logic fill:#ade8f4,stroke:#0077b6,stroke-width:2px,color:#000
    classDef db fill:#ffe8d6,stroke:#cb997e,stroke-width:2px,color:#000
    classDef core fill:#d8b4e2,stroke:#9c27b0,stroke-width:2px,color:#000

    Cron(("⏱️ Midnight CRON Job")):::trigger --> Engine["⚙️ Transaction Processing Engine"]:::logic
    
    Engine -- "1. Loops through Active Accounts" --> LedgerDB[("🐘 Aurora PostgreSQL:<br/>Accounts Table")]:::db
    Engine -- "2. Cross-references Mathematical Rates" --> Catalog[("🐘 Aurora PostgreSQL:<br/>Product_Catalog")]:::db
    
    Engine --> Check{"3. Evaluate Time Duration vs Account Age"}:::trigger
    
    Check -- "A. Not Matured Yet" --> Accrual["🧮 Execute Daily Accrual Algorithm"]:::logic
    Accrual -- "4a. Appends Nightly +Yield Entry" --> LedgerAPI["Internal: Core Ledger Service"]:::core
    
    Check -- "B. Lock Period Hit Maturity" --> Mature["🏁 Finalize Matured State"]:::logic
    Mature -- "4b. Commits Exact Maturity Payload" --> LedgerAPI
    Mature -- "5. Emits Domain Event" --> PubSub(("EventBridge:<br/>AccountMatured")):::trigger
    
    PubSub -. "6. Triggers Mandatory M30 Math" .-> Tax["Team 3: Tax Engine"]:::core
```
