# Core Ledger Service

## What is it?
The absolute source of truth for all money in the bank. This synchronous API is entirely isolated from the internet. It exists to guarantee that no account can ever have a negative balance, and following other accounting rules. 

## Core Logic & Rules
1. **Strict Double-Entry:** Every mutation must have corresponding credits and debits that balance perfectly to zero. 
2. **Immutable Append-Only:** You cannot edit or delete a transaction. You can only append a new inverse transaction to correct a mistake.
3. **No External Access:** This service cannot be reached by the web or mobile app.

## Data Flow Visualization
```mermaid
flowchart TD
    %% Colors
    classDef ext fill:#f9e1e1,stroke:#d62828,stroke-width:2px,color:#000
    classDef internal fill:#e2ece9,stroke:#2a9d8f,stroke-width:2px,color:#000
    classDef engine fill:#ade8f4,stroke:#0077b6,stroke-width:2px,color:#000
    classDef db fill:#ffe8d6,stroke:#cb997e,stroke-width:2px,color:#000
    classDef event fill:#fff3b0,stroke:#e9c46a,stroke-width:2px,color:#000

    User["👤 Mobile / Web App"]:::ext --> BFF["Deposits BFF"]:::engine
    BFF -- "1. Fetches Latest Cached Balance" --> ReadDB[("🐘 Materialized Read View:<br/>Customer_Balance_View")]:::db
    
    Onboarding["Team 1: Onboarding Service"]:::internal -- "2a. Submits Creation Request" --> Logic["⚙️ Core Ledger API"]:::engine
    Payments["Team 3: Payments Service"]:::internal -- "2b. Submits Withdrawal Request" --> Logic
    
    Logic -- "3. Validates Double-Entry & Funds Balance" --> ReadDB
    Logic -- "4. Strictly Appends PYO/PYI Immutable Row" --> WriteDB[("🐘 Core Ledger Engine:<br/>Ledger_Events")]:::db
    
    WriteDB -- "5a. Flushes CDC Stream (PostgreSQL WAL)" --> Proj["🧮 Materialized Projection Engine"]:::engine
    WriteDB -- "5b. Emits State Mutation Event" --> EventBus(("Central EventBridge PubSub")):::event
    
    Proj -- "6. Re-calculates and Updates User Balance" --> ReadDB
```
