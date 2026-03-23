# Architecture Flow Diagrams (Assignment Requested)
 
This document outlines the step-by-step sequence and process diagrams for the two primary architectural flows requested in the assignment:
 
 1. **Onboarding Journey** (From draft application to fully verified and active user)
 2. **Customer Experience Journey** (Checking balances, making withdrawals)
 
 ---
 
 ## 1. Onboarding Sequence Diagram (KYC, IDV, and Decisioning)
 
 The onboarding process is owned entirely by **Team 1 (Onboarding & Compliance)**, with final account provisioning handed off asynchronously to the rest of the bank via AWS EventBridge.
 
 ```mermaid
 sequenceDiagram
     actor User
     participant WebApp as Customer Web App
     
     box "Team 1: Onboarding"
         participant RegAPI as Registration API
         participant Webhook as IDV Webhook Handler
         participant Decision as Underwriting Engine
     end
     
     participant Onfido as External IDV Provider
     participant EventBridge as AWS EventBridge
     
     box "Downstream Services"
         participant Ledger as Core Ledger (Team 2)
         participant Auth as Auth Service (Team 4)
     end
 
     %% Phase 1: Application Draft
     User->>WebApp: Enter Personal Data & KYC Questionnaire
     WebApp->>RegAPI: POST /v1/applications
     RegAPI-->>WebApp: 201 Created (Status: DRAFT)
     
     %% Phase 2: Identity Verification & SDK Initialization
     WebApp->>RegAPI: Request IDV SDK Token
     RegAPI->>Onfido: POST /initiate-sdk-session Server-to-Server
     Onfido-->>RegAPI: Returns Secure SDK Token
     RegAPI-->>WebApp: Returns SDK Token
     
     WebApp->>Onfido: Native SDK Flow (Scan ID & Face)
     Onfido-->>WebApp: Success Callback
     WebApp->>RegAPI: PATCH /v1/applications/{id} (Submit Session)
     RegAPI-->>WebApp: 200 OK (Status: PENDING_IDV)
     
     note over Onfido, Webhook: Asynchronous Verification Process
     Onfido--)Webhook: POST Webhook (Verification Approved)
     Webhook->>Webhook: Validate HMAC Signature
     Webhook->>RegAPI: Update Application Status (IDV_APPROVED)
     
     %% Phase 3: Final Decision
     RegAPI--)Decision: Evaluate Risk & AML Rules
     Decision-->>RegAPI: Decision: APPROVED
     RegAPI->>RegAPI: Update Application Status (APPROVED)
     
     %% Phase 4: Account Provisioning (Event-Driven)
     RegAPI--)EventBridge: Publish Event: CustomerVerified
     
     EventBridge--)Ledger: Consume Event
     Ledger->>Ledger: Provision Empty Savings Account
     
     EventBridge--)Auth: Consume Event
     Auth->>Auth: Provision Identity Profile & Send Emails
 
     %% Phase 5: Polling / Refresh
     WebApp->>RegAPI: GET /v1/applications/{id}
     RegAPI-->>WebApp: 200 OK (Status: APPROVED, accounts provisioned)
 ```
 
 ---
 
 ## 2. Customer Experience Sequence Diagram (BFF, Balances, & Withdrawals)
 
 The customer experience combines reading data through the **GraphQL Federation Data Provider / BFF (Team 4)** to easily load the dashboard, and writing data (withdrawals) utilizing native REST and asynchronous payment clearing by **Team 3**.
 
 ```mermaid
 sequenceDiagram
     actor User
     participant WebApp as Customer Web App
     
     box "Team 2: Deposits"
         participant BFF as Customer Web BFF
         participant LedgerAPI as Core Ledger API
     end
     
     box "Team 3: Payments"
         participant Orch as Payout Orchestrator
         participant SEPA as Central Bank / Clearing
     end
 
     %% Flow A: Rendering the Dashboard
     note over User, SEPA: Scenario A: Loading the Dashboard (Balance & History)
     User->>WebApp: Login & Wait for Dashboard
     WebApp->>BFF: GET /v1/dashboard/summary (or GraphQL)
     
     par Fan-Out Queries
         BFF->>LedgerAPI: GET /v1/accounts/{id}/balance
         BFF->>LedgerAPI: GET /v1/accounts/{id}/transactions?limit=5
     end
     
     LedgerAPI-->>BFF: JSON Responses
     BFF->>BFF: Aggregate Dashboard Data
     BFF-->>WebApp: Unified Dashboard JSON
     WebApp-->>User: Render Dashboard
 
     %% Flow B: Initiating a Withdrawal
     note over User, SEPA: Scenario B: Initiating a Withdrawal to External Account
     User->>WebApp: Submit Payout Form (500 EUR to specific IBAN)
     
     WebApp->>LedgerAPI: POST /v1/accounts/{id}/withdrawals (Idempotency-Key)
     LedgerAPI->>LedgerAPI: Lock Row (SELECT FOR UPDATE)
     LedgerAPI->>LedgerAPI: Verify availableBalance >= 500
     LedgerAPI->>LedgerAPI: Deduct balance, change status to PROCESSING
     LedgerAPI-->>WebApp: 201 Created (Withdrawal Initiated)
     WebApp-->>User: Render "Transaction Pending" state
     
     %% Flow B Continued: Asynchronous Clearing Execution
     LedgerAPI--)Orch: Event: PayoutRequested (via Pub/Sub EventBus)
     Orch->>Orch: Validate external IBAN blacklist
     Orch->>SEPA: Transmit ISO-20022 CAMT / PAIN File
     SEPA-->>Orch: Accept PAIN.001 Batch
     
     %% Callback via Webhooks / Real-Time Updates (Optional WebSocket)
     note over Orch, LedgerAPI: Once settled, Ledger is updated to SETTLED.
     Orch--)LedgerAPI: Event: PayoutSettled
 ```
 
 ---
 
 ## 3. Extended Customer Capabilities: Adding Money (Top-Up)
 
 This sequence illustrates how a verified user adds more money (Top-Up) to their overnight savings account (M6). Because the bank relies on SEPA credit transfers, the Web App's primary role is to securely provide deposit instructions, while the backend asynchronous pipeline processes the clearing and alerts the user.
 
 ```mermaid
 sequenceDiagram
     actor User
     participant WebApp as Customer Web App 
     
     box "Team 2: Deposits"
         participant BFF as Customer Web BFF
         participant Ledger as Core Ledger API
     end
     
     box "Team 3: Payments"
         participant Parser as Inbound CAMT Parser
     end
     
     box "Team 4: Platform"
         participant Notify as Notification Engine
     end
     
     participant SEPA as Central Bank / SEPA
     
     %% Step 1: User requests deposit instructions
     User->>WebApp: Click "Add Funds"
     WebApp->>BFF: GET /v1/accounts/{id}/deposit-instructions
     BFF->>Ledger: Fetch linked IBAN & Reference
     Ledger-->>BFF: Returns Bank IBAN + Unique Reference Math
     BFF-->>WebApp: Renders Instructions UI
     WebApp-->>User: "Please wire funds to SEPA IBAN X with Ref Y"
     
     %% Step 2: User executes the transfer externally
     note over User, SEPA: User logs into External Bank and wires 1000 EUR
     User->>SEPA: Initiate External Wire Transfer
     
     %% Step 3: Asynchronous Clearing & Ledger Update
     SEPA->>Parser: Drop EOD CAMT.053 XML file
     Parser->>Parser: Parse file & Match Reference Y to Account
     Parser->>Ledger: POST /internal/v1/transactions (PYI: 1000 EUR)
     Ledger->>Ledger: Commit Transaction & Increment Balance
     
     %% Step 4: Notification to User
     Ledger--)Notify: Publish Event: DepositReceived
     Notify->>User: Sends SMS/Email: "You received 1000 EUR!"
     
     %% Step 5: User verifies updated balance
     User->>WebApp: Opens App
     WebApp->>BFF: GET /v1/dashboard/summary
     BFF-->>WebApp: Shows updated +1000 EUR balance
 ```
 
 ---
 
 ## 4. Financial Transaction Lifecycle (PYI, IBR, TAX, PYO)
 
 This details how the four primary financial transaction variants are programmatically generated and appended to the irreducible Core Ledger through asynchronous cron jobs, clearing files, and event-driven tax calculations.
 
 ```mermaid
 sequenceDiagram
     participant Bank as SEPA Clearing Bank
     
     box "Team 3: Payments"
         participant Parser as Inbound CAMT Parser
         participant Tax as Tax Engine
         participant Orch as Payout Orchestrator
     end
     
     box "Team 2: Deposits"
         participant Ledger as Core Ledger API
         participant Cron as Midnight Cron (Yield Engine)
     end
     
     %% 1. Incoming Funds (PYI - Pay In)
     note over Bank, Ledger: 1. Inbound Deposit (PYI)
     Bank->>Parser: SFTP Upload: Incoming CAMT.053 XML
     Parser->>Parser: Parse XML & Match IBAN
     Parser->>Ledger: POST /internal/v1/transactions (Type: PYI)
     Ledger->>Ledger: Increment Balance
     
     %% 2. Daily Interest Yield (IBR - Interest Bearing)
     note over Cron, Ledger: 2. Internal Yield Calculation (IBR)
     Cron->>Ledger: Trigger Daily Yield Engine (Midnight)
     Ledger->>Ledger: Calculate Yield based on APR
     Ledger->>Ledger: Append Transaction (Type: IBR)
     
     %% 3. Maturity Tax Withholding (TAX)
     note over Ledger, Tax: 3. Regulatory Taxation at Maturity (TAX)
     Ledger--)Tax: Event: AccountMatured (Pub/Sub)
     Tax->>Tax: Calculate 30% flat capital gains tax on Yield
     Tax->>Ledger: POST /internal/v1/transactions (Type: TAX)
     Ledger->>Ledger: Deduct Tax from Balance
     
     %% 4. Outgoing Funds (PYO - Pay Out)
     note over Ledger, Orch: 4. Final Payout Execution (PYO)
     Ledger--)Orch: Event: PayoutRequested
     Orch->>Bank: Dispatch Outbound PAIN.001 Batch (Type: PYO)
     Orch--)Ledger: Event: PayoutSettled (Set TX to Complete)
 ```
