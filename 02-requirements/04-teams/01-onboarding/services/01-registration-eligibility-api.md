# Registration & Eligibility API

## What is it?
This is the "front door" for new customers. When a user opens the Alborz Bank app and sign up.

## Core Logic & Rules
1. **Age Check:** Customers must be 18 or older to open a savings account.
2. **Residency Check:** Customers must reside in a legally approved region (e.g., Sweden).
3. **Draft Saving:** Customers can drop off and resume their application days later.
4. **Legal Compliance:** Before letting the user submit the final application, this API fetches the exact, legally required Terms & Conditions PDF from the Legal Document API and forces the user to accept it.

## The BFF Pattern (Backend-For-Frontend)
The Onboarding BFF acts as the orchestration layer. The client makes a single HTTP POST request, the BFF fetches the legal docs, and then delegates the raw PII exclusively to the heavily secured internal Registration API.


## Data Flow Visualization
```mermaid
flowchart TD
    %% Colors
    classDef client fill:#f9e1e1,stroke:#d62828,stroke-width:2px,color:#000
    classDef authGW fill:#ade8f4,stroke:#0077b6,stroke-width:2px,color:#000
    classDef bff fill:#ade8f4,stroke:#0077b6,stroke-width:2px,color:#000
    classDef core fill:#d8b4e2,stroke:#9c27b0,stroke-width:2px,color:#000
    classDef api fill:#fff3b0,stroke:#e9c46a,stroke-width:2px,color:#000
    classDef db fill:#ffe8d6,stroke:#cb997e,stroke-width:2px,color:#000
    classDef external fill:#a8dadc,stroke:#457b9d,stroke-width:2px,color:#000

    User["👤 Customer (App/Web)"]:::client --> Auth["🔐 Auth API "]:::authGW
    Auth -- "1. Logs in via BankID<br/>Issues OAuth Token" --> User
    
    User -- "2. Submits Base PII (Name, DOB)" --> Gateway["⚙️ API Gateway"]:::api
    Gateway --> BFF["λ Onboarding BFF"]:::bff
    
    BFF -- "3a. Fetches Regional T&C" --> Legal["📄 Legal Document API<br/>(Reads PDF from S3)"]:::external
    BFF -- "3b. Passes Auth & PII Payload" --> RegAPI["λ Core Registration API"]:::core
    
    RegAPI --> Check{"4. Check Constraints:<br/>Age >= 18? Regional Match?"}:::core
    
    Check -- "No" --> Reject["❌ Block Account Opening"]:::api
    Check -- "Yes" --> Save["5. Append Application Data"]:::core
    Save --> DB[("⚡ DynamoDB:<br/>Onboarding_Applications<br/>(Status = DRAFT)")]:::db
    
    DB -- "6. Returns Success" --> User
```
