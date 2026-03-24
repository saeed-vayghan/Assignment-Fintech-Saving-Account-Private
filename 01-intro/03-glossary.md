
## Glossary

This glossary defines key banking, compliance, and technical terms used throughout the Alborz Bank project.

### 1. Banking & Transactional Terms

| Term | Abbreviation | Description |
| :--- | :--- | :--- |
| **Savings Account** | - | A bank account for earning interest over time, not intended for daily transactions. |
| **Fixed Term Deposit** | - | A deposit locked for a specific duration in exchange for a guaranteed interest rate. |
| **Overnight Account** | - | A flexible savings account with no lock-in period and variable daily interest. |
| **Maturity** | - | The date when a fixed-term deposit reaches the end of its agreed period. |
| **Rollover** | - | Automatically renewing a fixed-term deposit for another term upon maturity. |
| **Interest Rate** | - | The cost of borrowing or reward for saving (Fixed or Variable). |
| **Accrued Interest** | - | Interest earned but not yet paid out to the customer. |
| **Withholding Tax** | - | Tax deducted by the bank from interest earnings before payout. |
| **Ledger** | - | An internal record-keeping system that tracks all financial transactions. |
| **Pay-In** | **PYI** | Inbound payment from an external bank to fund a deposit account. |
| **Pay-Out** | **PYO** | Outbound payment from Alborz Bank to a customer's external account. |
| **Cancellation** | **CAN** | Outbound payment resulting from early termination of a fixed-term deposit. |
| **Interest Booking** | **IBR** | internal ledger entry recording earned interest (non-cash movement). |
| **Tax Booking** | **TAX** | internal ledger entry recording withheld tax. |
| **IBAN** | - | International Bank Account Number used for identifying bank accounts globally. |
| **ACID** | - | Atomicity, Consistency, Isolation, Durability; standards for reliable transactions. |
| **Top-Up** | - | Adding funds to an existing flexible/overnight savings account. |
| **MTD** | - | Month-To-Date; refers to values accumulated since the start of the current month. |

---

### 2. Compliance & Legal Terms

| Term | Abbreviation | Description |
| :--- | :--- | :--- |
| **Know Your Customer** | **KYC** | Mandatory legal process to verify the identity and background of customers. |
| **Politically Exposed Person** | **PEP** | A person with a prominent public role, flagged for higher risk screening. |
| **Anti-Money Laundering** | **AML** | Regulations and procedures to prevent the generation of illegal income. |
| **Sanctions List** | - | Lists of individuals/nations subjected to financial restrictions (e.g., OFAC, EU). |
| **Identity Verification** | **IDV** | The process of proving a customer's identity via documents and biometrics. |
| **Onboarding** | - | The end-to-end process of registering and activating a new customer. |
| **Underwriting** | - | Evaluating customer eligibility based on residency, age, and risk criteria. |
| **PII** | - | Personally Identifiable Information; sensitive data protected by privacy laws (GDPR). |
| **RBAC** | - | Role-Based Access Control; restricting access based on user permissions. |

---

### 3. Technical & Infrastructure Terms

| Term | Abbreviation | Description |
| :--- | :--- | :--- |
| **Backend For Frontend** | **BFF** | Architecture pattern tailoring backends for specific frontend requirements. |
| **Infrastructure as Code** | **IaC** | Managing cloud infrastructure using machine-readable definition files. |
| **Cloud Development Kit** | **CDK** | AWS framework for defining infrastructure using modern programming languages. |
| **REST / GraphQL** | - | Architectural styles and protocols for building and querying APIs. |
| **ISO 20022** | - | Global standard for financial messaging and data exchange. |
| **Cash Management** | **CAMT** | ISO 20022 message format for account reporting and settlement files. |
| **SFTP** | - | Secure File Transfer Protocol used for sensitive file exchange with partners. |
| **EventBridge** | - | AWS serverless event bus for decoupled, event-driven service communication. |
| **SQS / DLQ** | - | Simple Queue Service and Dead Letter Queues for managing async task failures. |
| **Step Functions** | - | AWS orchestrator for complex, multi-step serverless workflows. |
| **Aurora (PostgreSQL)** | - | High-performance relational database used for the core transactional ledger. |
| **WAF** | - | Web Application Firewall; provides edge security and rate limiting for APIs. |

---

### 4. Integration & Tooling

| Tool / Provider | Category | Purpose |
| :--- | :--- | :--- |
| **BankID / Freja** | eID Provider | National electronic identity solutions for secure login. |
| **Onfido / Signicat** | IDV / KYC | Third-party services for identity proving and passport validation. |
| **Plaid / Tink** | Open Banking | Account ownership validation and real-time bank connectivity. |
| **ComplyAdvantage** | AML Screening | Global database for PEP, Sanctions, and adverse media screening. |
| **K6 / Artillery** | Load Testing | Tools for validating system performance and throughput. |
| **Jest / Cypress** | Testing | Frameworks for unit and end-to-end (E2E) automated testing. |
