# Alborz Bank — Database Models & Flows

This document visualizes the exact database schemas for the four engineering teams and maps out how data flows through them during key banking scenarios.

## 1. Unified Database Schema (ERD)

```mermaid
erDiagram
    %% ONBOARDING (DynamoDB)
    "ONBOARDING.Onboarding_Applications (DynamoDB)" {
        string PK_APPLICATION_UUID PK
        string GSI1_STATUS_SHARDED "Index"
        string GSI2_CUSTOMER_UUID "Index"
        uuid customer_id
        string status
        map personal_details
        map contact_details
        map identity_verification
        map bank_account_validation
        map pep_sanctions_screening
        map underwriting_decision
        list kyc_submission_versions
        list legal_agreements
        timestamp created_at
        timestamp updated_at
    }
    
    "ONBOARDING.IDV_Vendor_Webhooks (DynamoDB)" {
        string PK_APPLICATION_UUID PK
        string SK_TIMESTAMP_ID PK
        string GSI1_FAILED_STATUS_SPARSE "Index"
        string vendor_name
        string event_type
        string raw_payload
        string processed_status
        string failed_status
    }

    %% DEPOSITS (Aurora PostgreSQL)
    "DEPOSITS.Accounts (Aurora PostgreSQL)" {
        uuid account_id PK
        string account_number_iban UK "idx_account_number_iban"
        uuid customer_id FK "idx_customer_id"
        string account_type
        string currency
        string status "idx_status_maturity"
        string product_code
        decimal locked_interest_rate_pct
        int term_months
        timestamp maturity_date "idx_status_maturity"
        boolean auto_rollover
        timestamp created_at
        timestamp updated_at
        timestamp closed_at
    }

    "DEPOSITS.Ledger_Events (Aurora PostgreSQL)" {
        uuid event_id PK
        uuid account_id FK "idx_account_id_date"
        int sequence_number UK "idx_account_id_seq"
        string idempotency_key UK
        string transaction_type
        string direction
        string currency
        decimal amount
        decimal balance_after_event
        timestamp transaction_date "idx_account_id_date"
        timestamp settlement_date
        jsonb metadata
    }

    "DEPOSITS.Customer_Balance_View (Aurora PostgreSQL)" {
        uuid account_id PK
        uuid customer_id "idx_customer_id"
        decimal current_balance
        decimal available_balance
        decimal accrued_interest_mtd
        int last_event_sequence
        timestamp last_calculated_at
    }

    "DEPOSITS.Interest_Rates (Aurora PostgreSQL)" {
        uuid rate_id PK
        string product_code FK
        decimal annual_percentage_yield
        timestamp effective_from_date
        timestamp effective_to_date
        string created_by_admin_id
        timestamp created_at
    }

    "DEPOSITS.Product_Catalog (Aurora PostgreSQL)" {
        string product_code PK
        string product_name
        string account_type
        string currency
        decimal max_balance_limit
        boolean is_active
        timestamp created_at
    }

    %% PAYMENTS (DynamoDB)
    "PAYMENTS.Payout_Job_Locks (DynamoDB)" {
        string PK_PAYOUT_UUID PK
        string GSI1_FAILED_STATUS_SPARSE "Index"
        string failed_status
        string ledger_event_id
        string account_id
        amount number
        string destination_iban
        timestamp created_at
        timestamp updated_at
        number ttl
    }

    "PAYMENTS.Inbound_CAMT_Message_Locks (DynamoDB)" {
        string PK_MESSAGE_ID PK
        string GSI1_FAILED_STATUS_SPARSE "Index"
        string failed_status
        string file_name
        string payment_reference
        number amount
        number ttl
    }

    "PAYMENTS.Tax_Withholding_Log (DynamoDB)" {
        string PK_TAX_YEAR PK
        string SK_CUSTOMER_EVENT PK
        string GSI1_PENDING_REMITTANCE_SPARSE "Index"
        timestamp pending_remittance_date
        string customer_tax_id
        number interest_earned
        number tax_withheld
        number tax_rate_applied
    }

    %% PLATFORM (DynamoDB)
    "PLATFORM.Auth_Users (DynamoDB)" {
        string PK_USER_UUID PK
        string GSI1_NATIONAL_HASH "Index"
        string national_id_hash
        string customer_id
        list app_roles
        timestamp created_at
    }

    "PLATFORM.Auth_Tokens (DynamoDB)" {
        string PK_TOKEN_HASH PK
        string GSI1_USER_UUID "Index"
        string auth_identity_id
        string customer_id
        list app_roles
        number expires_at_ttl
    }

    "PLATFORM.Refresh_Tokens (DynamoDB)" {
        string PK_TOKEN_HASH PK
        string GSI1_USER_UUID "Index"
        string auth_identity_id
        string device_fingerprint
        timestamp expires_at
        boolean revoked
    }

    %% Relationships
    "ONBOARDING.Onboarding_Applications (DynamoDB)" ||--o{ "ONBOARDING.IDV_Vendor_Webhooks (DynamoDB)" : receives
    "ONBOARDING.Onboarding_Applications (DynamoDB)" ||--o{ "DEPOSITS.Accounts (Aurora PostgreSQL)" : opens
    "PLATFORM.Auth_Users (DynamoDB)" ||--|{ "ONBOARDING.Onboarding_Applications (DynamoDB)" : owns
    "PLATFORM.Auth_Users (DynamoDB)" ||--o{ "PLATFORM.Auth_Tokens (DynamoDB)" : authenticates
    "PLATFORM.Auth_Users (DynamoDB)" ||--o{ "PLATFORM.Refresh_Tokens (DynamoDB)" : authenticates
    "DEPOSITS.Accounts (Aurora PostgreSQL)" ||--o{ "DEPOSITS.Ledger_Events (Aurora PostgreSQL)" : mutates
    "DEPOSITS.Accounts (Aurora PostgreSQL)" ||--|| "DEPOSITS.Customer_Balance_View (Aurora PostgreSQL)" : projects_to
    "DEPOSITS.Product_Catalog (Aurora PostgreSQL)" ||--o{ "DEPOSITS.Accounts (Aurora PostgreSQL)" : instantiates
    "DEPOSITS.Product_Catalog (Aurora PostgreSQL)" ||--o{ "DEPOSITS.Interest_Rates (Aurora PostgreSQL)" : tracks_yield_history
    "DEPOSITS.Ledger_Events (Aurora PostgreSQL)" ||--o| "PAYMENTS.Payout_Job_Locks (DynamoDB)" : triggers
    "DEPOSITS.Accounts (Aurora PostgreSQL)" ||--|{ "PAYMENTS.Tax_Withholding_Log (DynamoDB)" : incurs
```

---

## 2. Scenario Data Flows

### Flow A: Sign Up & Identity Verification (Onboarding & Underwriting)
This sequence illustrates the state machine a new customer passes through to clear bank compliance constraints and receive final underwriting approval.

```mermaid
sequenceDiagram
    autonumber
    participant User as 👤 Customer
    participant Auth as 🔐 Platform (Auth)
    participant Dynamo as ⚡ DynamoDB (Applications)
    participant IDV as 🕵️ Third-Party IDV
    participant Engine as ⚙️ Underwriting Engine (SFN)
    participant Audit as 📝 ELK (Compliance Line)

    User->>Auth: Log in with National eID
    Auth->>User: Issue Auth Identity ID
    
    User->>Dynamo: Start Application (DRAFT)
    Dynamo-->>User: Create Application UUID
    
    User->>IDV: Submit Passport + Selfie
    IDV->>User: Synchronous UI Success
    
    IDV-->>Dynamo: Async Webhook (DOCUMENT_VERIFIED)
    Note over Dynamo: IDV_Vendor_Webhooks stores raw JSON
    
    Dynamo->>Audit: Publish Immutable KYC_SUBMITTED Event
    Dynamo->>Dynamo: Transition Status -> PENDING_UNDERWRITING
    
    Note over Dynamo, Engine: The final automated compliance decision
    Dynamo-->>Engine: Trigger: Application Ready
    Engine->>Engine: Rule 1: Validate IDV == VERIFIED
    Engine->>Engine: Rule 2: Validate PEP == NO_MATCH
    alt All rules pass mathematically
        Engine->>Dynamo: Update status -> ACTIVE (Approved)
    else Any rule fails hard
        Engine->>Dynamo: Update status -> REJECTED
    else Ambiguous Match / Edge Case
        Engine->>Dynamo: Update status -> PENDING_MANUAL_REVIEW
    end
```

---

### Flow B: Account Creation & Initial Deposit (Onboarding -> Deposits -> Payments)
This captures the moment the application is approved, the bank account is physically provisioned in the ledger, and the user sends in their very first wire transfer.

```mermaid
sequenceDiagram
    autonumber
    participant Onboarding as 📋 Target (Onboarding)
    participant Deposits as 🏦 Aurora (Ledger)
    participant Payments as 💸 λ CAMT Parser
    participant UserBank as 🏦 External Bank
    
    Note over Onboarding: Application marked ACTIVE
    Onboarding->>Deposits: POST /accounts (Create Overnight)
    Deposits->>Deposits: INSERT INTO Accounts (Generate IBAN)
    Deposits-->>Onboarding: Account UUID & Account IBAN
    
    Note over UserBank, Payments: User performs external bank wire transfer
    UserBank->>Payments: SFTP Upload (CAMT)
    Payments->>Deposits: POST /internal/deposits (PYI) targeting Account IBAN
    Deposits->>Deposits: INSERT INTO Ledger_Events
    Deposits->>Deposits: Async Update Customer Balance
    Deposits-->>Payments: 201 Created (Idempotent)
```

---

### Flow C: Early Term Cancellation & Withdrawal (Deposits -> Payments)
This flow tracks the complex distributed transaction of breaking a time-locked deposit and returning the money to the user.

```mermaid
sequenceDiagram
    autonumber
    participant App as 📱 Web UI
    participant Deposits as 🏦 Aurora (Ledger)
    participant Bus as 🚌 EventBridge
    participant Payments as 💸 DynamoDB (Payout Jobs)
    participant Bank as 🏦 Partner Bank
    
    App->>Deposits: POST /accounts/:id/cancel
    Deposits->>Deposits: Verify Account Type = FIXED_TERM
    Deposits->>Deposits: INSERT INTO Ledger_Events (CAN)
    Note over Deposits: Ledger records negative withdrawal amount (minus penalties)
    Deposits->>Bus: Publish Domain Event (`PayoutRequested`)
    
    Bus-->>Payments: Async Trigger
    Payments->>Payments: INSERT Payout_Job_Locks (Status: QUEUED)
    Note over Payments: DynamoDB guarantees no double-generation
    
    Payments->>Bank: Dispatch SEPA/SWIFT Instruction API
    Bank-->>Payments: Instruction Accepted HTTP 200
    Payments->>Payments: UPDATE Payout_Job_Locks (Status: PROCESSING)
```

---

### Flow D: Inbound Payment (CAMT Import to Core Ledger)
This sequence maps how the Payments team converts raw external XML into a strictly ACID-compliant PostgreSQL ledger.

```mermaid
sequenceDiagram
    autonumber
    participant CentralBank as 🏦 Partner Bank
    participant S3 as 🗄️ S3 (Settlement Archive)
    participant Dynamo as ⚡ DynamoDB (Inbound Locks)
    participant Payments as ⚙️ λ CAMT Parser
    participant Deposits as ⚙️ λ Core Ledger API
    participant Aurora as 🐘 Aurora (Ledger_Events)
    participant View as 🐘 Aurora (Customer_Balance)

    CentralBank->>S3: SFTP Upload: camt053_0900.xml
    S3-->>Payments: S3 ObjectCreated Event
    Payments->>Dynamo: Write Lock (MessageID)
    Note over Dynamo: Idempotency Check (Reject if exists)
    Payments->>Payments: Parse XML to JSON
    Payments->>Deposits: POST /internal/deposits (PYI)
    Deposits->>Aurora: INSERT INTO Ledger_Events (Amount: +$50)
    Note over Aurora: Event Sourcing Append
    Aurora-->>View: Async CDC Trigger (Update Balance)
    Deposits-->>Payments: 201 Created
    Payments->>S3: Tag S3 Object (processed_status=SUCCESS)
```

---

### Flow E: Midnight Account Maturity (Fixed-Term Rollover)
This sequence maps the scheduled domain logic for migrating money states.

```mermaid
sequenceDiagram
    autonumber
    participant Cron as 🕐 EventBridge (Midnight)
    participant MaturitySvc as ⚙️ λ Maturity Job (Deposits)
    participant Aurora as 🐘 Aurora (Accounts + Ledger)
    participant Bus as 🚌 EventBridge (Central)
    participant TaxSvc as ⚙️ λ Tax Engine (Payments)
    participant TaxDB as ⚡ DynamoDB (Tax Log)

    Cron->>MaturitySvc: Trigger Daily Run
    MaturitySvc->>Aurora: SELECT * FROM Accounts WHERE status='ACTIVE' AND maturity_date=TODAY
    
    alt If auto_rollover == true
        MaturitySvc->>Aurora: UPDATE Accounts SET maturity_date = +Term
    else If auto_rollover == false
        MaturitySvc->>Aurora: UPDATE Accounts SET status='MATURED'
        MaturitySvc->>Aurora: INSERT INTO Ledger_Events (IBR + Interest)
        MaturitySvc->>Bus: Publish Domain Event (`AccountMatured`)
        
        Bus-->>TaxSvc: Subscribed Event Payload
        TaxSvc->>TaxSvc: Calculate 30% Withholding Rate
        TaxSvc->>TaxDB: Record Liability (Tax_Withholding_Log)
        TaxSvc->>MaturitySvc: POST /internal/deposits (TAX deduct)
    end
```