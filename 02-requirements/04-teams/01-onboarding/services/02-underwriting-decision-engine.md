# Underwriting & Decision Engine

## What is it?
This is the automated "brain" of the bank's compliance team.

## Core Logic & Rules
1. **Wait for Evidence:** It pauses until all external checks (like the ID photo validation and the anti-money laundering watchlists) are completely finished.
2. **The "Green Light" Rule:** If the ID is perfectly `VERIFIED` AND the customer has `NO_MATCH` on external blacklists, the engine approves the application.
3. **The "Red Light" Rule:** If the ID is proven to be a fake or expired, it instantly rejects the applicant.
4. **The "Yellow Light" Rule:** If the person has the exact same name as a sanctioned politician, the engine freezes the application as `PENDING_MANUAL_REVIEW` and alerts a human compliance officer to take over.

## Detailed Execution Steps
The Step Function executes the following distinct stages in order:
1. **Trigger Phase:** Wakes up via EventBridge when the `All_KYC_Collected` event fires.
2. **Data Aggregation:** Queries DynamoDB to pull the user's PII, the exact IDV vendor results, and the AML screening payload.
3. **Hard Rules Check:** Instantly branches to `REJECTED` if Age < 18 or Residence is outside supported zones (redundancy check).
4. **Vendor Evaluation:** Asserts that the IDV Webhook explicitly returned `VERIFIED`. Any `SUSPECTED_FRAUD` instantly routes to rejection.
5. **Watchlist Evaluation:** Asserts that the ComplyAdvantage AML payload shows `NO_MATCH`. A partial PEP match routes to human review.
6. **Final State Commitment:** Writes the final boolean `APPROVED` or `REJECTED` state back to DynamoDB, locking the application.
7. **Downstream Notification:** Publishes the `CustomerVerified` event back to the global EventBus for other teams (like Team 2 Core Ledger) to react and physically open the bank account.

## Data Flow Visualization
```mermaid
flowchart TD
    %% Colors
    classDef hook fill:#e2ece9,stroke:#2a9d8f,stroke-width:2px,color:#000
    classDef sfn fill:#fff3b0,stroke:#e9c46a,stroke-width:2px,color:#000
    classDef decision fill:#ade8f4,stroke:#0077b6,stroke-width:2px,color:#000
    classDef manual fill:#f9e1e1,stroke:#d62828,stroke-width:2px,color:#000
    classDef db fill:#ffe8d6,stroke:#cb997e,stroke-width:2px,color:#000

    VendorHook["Webhook: Signicat/Onfido<br/>IDV Completed"]:::hook --> SFN["1. AWS Step Function<br/>Triggered via EventBridge"]:::sfn
    VendorHook2["Webhook: ComplyAdvantage<br/>PEP/Sanctions Checked"]:::hook --> SFN
    
    SFN --> Fetch["2. Fetch Application Data"]:::sfn
    Fetch --> DB[("⚡ DynamoDB<br/>(Reads Current DRAFT State)")]:::db
    
    DB --> CheckIDV{"3. Assess IDV Vendor Result"}:::sfn
    CheckIDV -- "SUSPECTED_FRAUD" --> Reject["❌ Reject: Fraudulent ID"]:::decision
    
    CheckIDV -- "VERIFIED" --> CheckAML{"4. Assess AML/PEP Match"}:::sfn
    CheckAML -- "EXACT_MATCH" --> Reject
    
    CheckAML -- "PARTIAL_MATCH" --> Flag["⚠️ Freeze: PENDING_MANUAL_REVIEW"]:::manual
    Flag -- "Alerts Team Console" --> Officer["Human Compliance Officer"]:::manual
    
    CheckAML -- "NO_MATCH" --> Approve["✅ Approve: Status = VERIFIED"]:::decision
    
    Approve -- "5. Commits Final State" --> DBWrite[("⚡ DynamoDB<br/>(Update record)")]:::db
    DBWrite -- "6. Broadcasts Central Event" --> Bus(("EventBridge Pub/Sub:<br/>CustomerVerified Event")):::hook
```
