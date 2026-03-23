# Auth-Service

## What is it?
The central identity and access management (IAM) primitive for the entire Alborz Bank B2C architecture. It abstracts away complex federated identity protocols (like Swedish BankID OIDC) and provides a unified, secure Opaque Token issuance system. It guarantees that domain teams (like Deposits or Onboarding) never have to write custom authentication logic or securely hash passwords.

## Core Logic & Rules
1. **Centralized Authentication, Decentralized Authorization:** The Auth-Service proves *who* the user is, but the downstream microservices decide *what* that user is allowed to do.
2. **Stateless Edge, Stateful Core:** Access tokens are short-lived (15 minutes) and opaque. The API Gateway validates them statefully against DynamoDB on every request to ensure immediate global revocation capabilities (e.g., locking a stolen phone).
3. **Pii Minimization:** Core domain microservices use internal `customer_id` UUIDs. The Auth-Service handles the mapping from sensitive National IDs (PNR) to these internal agnostic IDs.

## Data Flow Visualization
```mermaid
flowchart TD
    %% Colors
    classDef ext fill:#f9e1e1,stroke:#d62828,stroke-width:2px,color:#000
    classDef internal fill:#e2ece9,stroke:#2a9d8f,stroke-width:2px,color:#000
    classDef engine fill:#e2ece9,stroke:#2a9d8f,stroke-width:2px,color:#000
    classDef db fill:#c0efd1,stroke:#264653,stroke-width:2px,color:#000

    User["👤 Mobile / Web App"]:::ext -- "1. POST /auth/login" --> APIGW["⚙️ API Gateway"]:::internal
    APIGW --> AuthLogic["🔐 Auth-Service Lambda"]:::engine
    
    AuthLogic -- "2. Initiates OIDC Flow" --> BankID(("🇸🇪 Swedish BankID")):::ext
    BankID -- "3. Returns Valid Identity (PNR)" --> AuthLogic
    
    AuthLogic -- "4. Hashes PNR & Fetches Customer UUID" --> DB1[("⚡ DynamoDB:<br/>Auth_Users")]:::db
    
    AuthLogic -- "5. Generates Secure Tokens" --> AuthLogic
    
    AuthLogic -- "6a. Stores short-lived Access Token" --> DB2[("⚡ DynamoDB:<br/>Auth_Tokens")]:::db
    AuthLogic -- "6b. Stores long-lived Refresh Token" --> DB3[("⚡ DynamoDB:<br/>Refresh_Tokens")]:::db
    
    AuthLogic -- "7. Returns Tokens" --> User
```
