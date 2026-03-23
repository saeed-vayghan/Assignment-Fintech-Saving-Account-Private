# Team 1: Onboarding & Compliance

### **Purpose**:
Manage the end-to-end customer registration flow from the moment a user clicks "Sign Up". This team is responsible for ensuring the bank legally and securely validates the identity of every applicant before they are allowed to open a savings account.

### **UI Applications**:
1. **Customer Registration Web App:** The public-facing wizard guiding users through data input, document uploads, KYC questions (M4), and legal agreements like T&C/Privacy Policies (M38).
2. **Compliance Officer Dashboard:** An internal back-office tool (S2) used to manually review PEP/Sanctions matches (M3), verify edge-case KYC documents, and formally approve or reject accounts.
3. **Customer Support Tooling:** Internal read-only CRM dashboards (S5) for support agents to view a customer's onboarding journey state and identify drop-off points without seeing raw PII.

### **User-Facing Service**:
Yes. The Customer Registration Web App and its accompanying APIs are the absolute first touchpoint for every new user of the bank, capturing initial data (M1).

### **Authentication / Authorization Mechanisms**:
*(See `06-auth-service.md` for Centralized Issuance / Decentralized AuthZ strategy)*
- **Customer Registration Web App (UI):** Relies on `eID` (BankID) or `Local-Cred` for AuthN, brokered by the Platform team. Uses the **Customer Web BFF** for data aggregation.
- **Registration & Eligibility API:** The API Gateway validates the `OAuth-Token` (AuthZ) before yielding to local business logic.
- **Compliance Dashboards & Support CRM:** Relies on Corporate `SSO` AuthN. Integrated via the **Back-office Operations BFF**.
- **Compliance & Support APIs:** Requires strict `OAuth-RBAC` (`Role: Compliance_Officer` or `Role: Support_Agent`).
- **KYC & IDV Webhooks:** Authenticated inbound via `API-Key` and rigid payload HMAC validation.
- **Bank Validation (Tink):** Authenticated outbound via `API-Key` (Client Credentials).

### **Microservices**: (Serverless / AWS Lambda)
  1. `Registration & Eligibility API`: Enforces age/residency rules (M25), tracks the `DRAFT → PENDING_VERIFICATION → ACTIVE` state machine (M28), and presents Legal Documents (M38).
  2. `KYC & IDV Webhook Service`: Instantly pushes all incoming vendor webhooks into an SQS FIFO queue to guarantee zero data loss (M39). Processors natively consume from this queue. Built with a vendor fallback strategy (S8).
  3. `Compliance & AML Service`: Manages PEP/Sanctions screening logic (M3) and asynchronous AML suspicious transaction monitoring (M31).
  4. `Bank Validation Service`: Interfaces with open banking APIs to verify the user owns the IBAN they provided for future payouts (M2).
  5. `Underwriting & Decision Engine`: An autonomous AWS Step Function that listens for the "All Data Collected" event, runs strict automated compliance math across all vendor inputs, and outputs the final verifiable `APPROVED` or `REJECTED` decision.
  6. `Legal Document API`: Serves immutable, versioned PDFs (T&C, Privacy Policy) from S3 based on the user's specific region and account type (M38).

### **Databases & Storage Access**:

| Database / Storage | Type | Access Level | Purpose / Justification |
|--------------------|------|--------------|-------------------------|
| **Amazon DynamoDB** (Onboarding State) | NoSQL Key-Value | **Read / Write** (Owned) | **Flexible Schema:** Changing JSON docs (KYC, vendor webhooks).<br>**Speed & Scale:** Handles massive registration spikes.<br>**State Machine:** Fast lookups for `DRAFT` → `VERIFIED` → `ACTIVE`. |
| **Amazon S3** (Legal Assets) | Object Storage | **Read / Write** (Owned) | **Immutable History:** Stores explicitly versioned PDF legal agreements presented to users before signing. |
| **ELK Stack** (Central Logging) | Search Index | **Write** (via Platform API) | **Cross-Team Dependency:** All compliance actions, PEP flags, and API events are pushed to the central observability cluster owned by **Team 4 (Platform)**. |

### **Caching Layer**:
No dedicated caching needed; DynamoDB key lookups are sufficient for state tracking and questionnaire definitions.

### **Message Queue / Async Comm**:
- Uses `AWS SQS (FIFO)` immediately upon receiving IDV webhooks to guarantee zero payload loss during internal outages (M39).
- Uses `AWS EventBridge` to asynchronously trigger the **Underwriting AWS Step Function** the exact moment a customer's `DRAFT` state reaches `PENDING_UNDERWRITING`.
- Uses `AWS EventBridge` to publish a `CustomerVerified` event once the Underwriting Engine formally approves the account, notifying the Deposits team that the customer is ready for a ledger account.
- Uses `AWS EventBridge` to subscribe to the `TransactionCommitted` domain event published by the Deposits team
  - enabling asynchronous AML Suspicious Transaction Monitoring (M31) without blocking real-time database transactions.

### **External Systems**:
General User Browsers/Mobile clients hitting the Registration Web App.

### **Third-Party Services**:
  - Signicat / Swedish BankID / Onfido / IDnow (Identity Verification / Liveness Checks) (M1)
  - ComplyAdvantage (AML, PEP, Sanctions screening) (M3)
  - Plaid / Tink (Open Banking account validation) (M2)
  - Customer support CRM API (S5)
  *Note: Integrations are built behind a facade pattern to support the Vendor Fallback Strategy (S8).*

### **Other Team Interactions**:
  - **Platform Team**: Leverages their Auth-Service for authentication and KMS for encrypting all captured PII at rest.
  - **Deposits Team**: The Deposits team listens to Onboarding's `CustomerVerified` event to create the actual financial ledger accounts.

### **Edge Cases & Failure Scenarios**:

| Category | Scenario | Immediate Strategy / Impact | HITL Action / Threshold |
|----------|----------|-----------------------------|-------------------------|
| **Retryable** | **Third-Party API Downtime:** 5xx errors from Signicat/Plaid/ComplyAdvantage. | Exponential backoff. | **Platform Ops (Major):** If >3 failures, alert ops to trigger Vendor Fallback (S8). |
| **Retryable** | **Webhook Delivery Failure:** Vendor misses our DynamoDB via network drop. | Vendor system auto-retries payload delivery. | **On-Call Engineer (Moderate):** Inspects the Dead Letter Queue (DLQ) to unblock the user. |
| **Moderate** | **Unclear Document / Glare:** IDV cannot process user document. | Automated flow halts. | **Customer Support:** Reviews image in CRM and prompts user to retake photo. |
| **Moderate** | **Session Expiry Mid-Flow:** User leaves KYC tab open for >4 hours. | Gracefully forces re-authentication. | **Support Agents:** Can view exact drop-off in CRM (S5) if user complains. |
| **Major** | **PEP/Sanctions Match:** User hits international watchlist or PEP database. | Hard stop on automation. | **Compliance & Legal:** Jointly investigate and formally record rejection in audit logs (M3). |
| **Major** | **Suspected Fraud / Velocity Attack:** 50+ signups from same IP in 5 mins. | Registration API proactively blocks IP/fingerprint. | **AML/Security Team:** Investigates attack and scrubs pending accounts before ledger creation. |

### **Logs and Metrics**:
- **Audit Trails:** Immutable logs recording exactly who approved what, and when T&Cs/Privacy policies were accepted (M38).
- **Compliance Logs:** Detailed records for AML suspicious transaction monitoring and regulatory exports (M31).
- **Product Metrics:** Funnel drop-off metrics (e.g., Mixpanel) heavily utilized by product and support tools (S5).

### **API Exposed**:
  - External REST APIs (Customer Registration flow)
  - Internal REST APIs (Compliance & Support Dashboards)
  - Webhook endpoints (Receiving vendor callbacks)

### **Roadmap & MoSCoW Prioritization References**:
#### **Must Have (MVP)**
  - `M1`: Customer registration & identity verification
  - `M2`: Bank account validation (IBAN + ownership)
  - `M3`: PEP/Sanctions screening with compliance review
  - `M4`: KYC questionnaire (5 questions, configurable)
  - `M25`: Customer eligibility validation (age ≥ 18, residency, entity type)
  - `M28`: Customer state machine (DRAFT → PENDING_VERIFICATION → ACTIVE / REJECTED)
  - `M31`: Regulatory reporting: AML suspicious transaction monitoring
  - `M38`: Legal documents: Terms & Conditions, privacy policy, fee schedule presented during onboarding
  - `M39`: KYC Webhook FIFO Queue: Immediate ingestion of IDV webhooks to SQS FIFO to prevent data loss

#### **Should Have**
  - `S2`: Compliance officer dashboard (review PEP flags, approve/reject)
  - `S5`: Customer support tooling: view customer journey, identify drop-off points
  - `S8`: Vendor fallback strategy: alternative providers if primary IDV/PEP/bank validation provider is down
