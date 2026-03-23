# 🚀 Unified Data Provider Strategy (GraphQL BFF)
 
 **Do we need a unified API Data Provider (like a GraphQL Gateway or Backend-for-Frontend)?**
 
 _Short Answer:_ **Yes.** For client-facing applications (the Customer Web App and the Internal Admin Portal), a unified data provider is highly recommended. However, for internal service-to-service communication, it should be avoided.
 
 ---
 
 ## The Problem: Chatty Microservices
 In the Alborz Bank architecture, data is strictly isolated by domain:
 - **Team 1 (Onboarding):** Owns KYC status, Legal agreements, and Identity.
 - **Team 2 (Deposits):** Owns account balances and transaction histories.
 - **Team 3 (Payments):** Owns pending payouts and accrued taxes.
 
 If the React frontend wants to render a simple "Dashboard" showing the user their KYC status, their current balance, and their last 5 transactions, the frontend would have to make **three separate REST API calls** over the public internet, handle three potential network failures, and stitch the data together in the browser.
 
 ## ✅ The Solution: A GraphQL Data Provider (BFF)
 We should introduce a **Data Provider Service** (acting as a Federation Gateway or BFF - Backend-for-Frontend) sitting behind the API Gateway.
 
 ### Architecture Flow
 
 ```mermaid
 flowchart TD
     classDef client fill:#bde0fe,stroke:#023e8a,stroke-width:2px,color:#000
     classDef bff fill:#ffafcc,stroke:#a2d2ff,stroke-width:2px,color:#000
     classDef api fill:#ade8f4,stroke:#0077b6,stroke-width:2px,color:#000
     classDef db fill:#c0efd1,stroke:#264653,stroke-width:2px,color:#000
     
     WebApp["Customer Web App<br/>(React/Next.js)"]:::client
     AdminUI["Back-Office Portal<br/>(Admin Dashboard)"]:::client
     
     subgraph Federation Layer
         GraphQL("Unified GraphQL Data Provider / BFF"):::bff
     end
     
     subgraph Domain Microservices ["Isolated REST APIs"]
         OnboardingAPI("Team 1: Onboarding API<br/>KYC & State"):::api
         DepositsAPI("Team 2: Deposits API<br/>Balances & Ledgers"):::api
         PaymentsAPI("Team 3: Payments API<br/>Payouts & Taxes"):::api
     end
     
     WebApp -- "Single GraphQL Query" --> GraphQL
     AdminUI -- "Single GraphQL Query" --> GraphQL
     
     GraphQL -- "Resolver Fan-Out" --> OnboardingAPI
     GraphQL -- "Resolver Fan-Out" --> DepositsAPI
     GraphQL -- "Resolver Fan-Out" --> PaymentsAPI
     
     OnboardingAPI -.-> DB1[("DynamoDB")]:::db
     DepositsAPI -.-> DB2[("Aurora DB")]:::db
     PaymentsAPI -.-> DB3[("DynamoDB")]:::db
 ```
 
 ### Why It's Powerful:
 1. **Prevents Over-fetching & Under-fetching:** A GraphQL query allows the frontend to request exactly what it needs in a single network trip:
    ```graphql
    query GetDashboard {
      customer(id: "123") {
        kycStatus
        accounts {
          balance { amount currency }
          recentTransactions(limit: 5) { type amount date }
        }
      }
    }
    ```
 2. **Backend Decoupling:** The internal REST APIs (`01-api-rules.md`) remain strict and domain-driven. The GraphQL resolvers handle the complex fan-out to the internal microservices (Deposits API, Onboarding API) and aggregate the JSON responses.
 3. **Centralized Security & Caching:** The data provider acts as a choke point to enforce rate-limiting, validate JWTs, and cache frequently requested, slow-changing data (like Interest Rates) using Redis/ElastiCache before hitting the core databases.
 