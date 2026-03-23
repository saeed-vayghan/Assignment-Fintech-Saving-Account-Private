# 🛡️ Alborz Bank — Architectural Threat Models

Threat modeling is how we identify and mitigate security vulnerabilities _before_ writing code. For Alborz Bank, we utilize the Microsoft **STRIDE** methodology (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege) combined with a Zero-Trust Architecture. 

This document maps out the specific attack vectors and architectural defenses for four critical microservices across the bank.

---

## 🛂 Team 1 (Onboarding): IDV Webhook Handler
**The Scenario:** A third-party provider (e.g., Onfido, Signicat) scans a customer's passport and sends an asynchronous webhook payload to our API to confirm they are a real person and have cleared AML (Anti-Money Laundering).

### 🔴 The Attack Vectors
| Threat Category | Specific Attack Vector | Severity |
| :--- | :--- | :--- |
| **Spoofing** | A malicious user discovers our hidden Webhook URL and manually sends a `{"status": "APPROVED", "applicant": "John Doe"}` payload to magically bypass KYC constraints. | 🔴 CRITICAL |
| **Tampering** | A Man-in-the-Middle (MitM) intercepts a legitimate webhook routing through the internet and maliciously swaps a `REJECTED_FRAUD` status to `APPROVED`. | 🔴 CRITICAL |
| **Denial of Service** | A botnet floods the webhook endpoint with 10,000 req/sec to overwhelm the backend database and prevent legitimate customers from finishing their onboarding journey. | 🟠 HIGH |

### 🟢 The Architectural Defenses
> [!TIP]
> **Defense-in-Depth applied here:**
> 1. **HMAC Cryptographic Signatures (Spoofing / Tampering):** The API Gateway enforces a custom `X-Signature` header. The payload is hashed using a shared secret only the IDV vendor knows. If the SHA-256 hash doesn't match perfectly, AWS WAF drops the request (401 Unauthorized) before it ever wakes up a Lambda function.
> 2. **SQS FIFO Buffer Zones (DoS):** Verified webhooks are never written straight to the database. They are dumped into an Amazon SQS Queue. The queue absorbs any massive internet spikes, allowing our backend to process them at a mathematically safe, throttled concurrency limit limit.

---

## 🏦 Team 2 (Deposits): Core Ledger API
**The Scenario:** The synchronous core engine of the bank. It accepts requests to move money *out* of an account (a Payout / Withdrawal request) and appends it to the immutable ledger.

### 🔴 The Attack Vectors
| Threat Category | Specific Attack Vector | Severity |
| :--- | :--- | :--- |
| **Elevation of Privilege** | An authenticated user (`user_A`) successfully logs in, but modifies the JSON payload to request a withdrawal from `user_B`'s account ID. | 🔴 CRITICAL |
| **Tampering (Race Conditions)** | A user holds a $100 balance. They deploy a script to simultaneously execute two $100 withdrawals at the exact same millisecond. The database reads $100 twice, approves both, and settles at a mathematically corrupted -$100 balance. | 🔴 CRITICAL |

### 🟢 The Architectural Defenses
> [!IMPORTANT]
> **Defense-in-Depth applied here:**
> 1. **Decentralized Authorization (Privilege):** The API Gateway extracts the `customer_id` from the secure auth token and injects it as a mandatory HTTP header. When the Core Ledger queries the Aurora database, it hardcodes `WHERE account_id = request.id AND owner_id = header.customer_id`. The database explicitly rejects the cross-account theft attempt.
> 2. **Optimistic Concurrency Locking (Tampering):** The database enforces a strict `UNIQUE CONSTRAINT (account_id, sequence_number)` on ledger events. The second simultaneous withdrawal will crash into the constraint, rollback its transaction, fetch the new $0 balance, and correctly reject the withdrawal as insufficient funds.

---

## 💸 Team 3 (Payments): Payout Job Orchestrator
**The Scenario:** Responsible for physically routing millions of dollars out of Alborz Bank by transmitting fiat clearing instructions (SEPA/SWIFT) to the Central Bank's API or SFTP servers.

### 🔴 The Attack Vectors
| Threat Category | Specific Attack Vector | Severity |
| :--- | :--- | :--- |
| **Spoofing (Internal)** | A compromised internal microservice (e.g., the Notification Engine) tries to spoof an internal call to the Payout Orchestrator to steal fiat currency. | 🔴 CRITICAL |
| **Repudiation** | An outbound wire transfer is processed flawlessly, but a few months later the Central Bank claims we never sent the clearing file, leading to massive financial fines and balance discrepancies. | 🟠 HIGH |
| **Tampering** | A transient network error between AWS and the Central Bank results in the API "timing out." The Orchestrator resends the payload blindly, accidentally double-paying out a $50,000 withdrawal. | 🔴 CRITICAL |

### 🟢 The Architectural Defenses
> [!NOTE]
> **Defense-in-Depth applied here:**
> 1. **AWS IAM Machine-to-Machine Auth (Spoofing):** Microservices do not use passwords. They use cryptographic IAM execution roles. Only the exact ARN of the `Deposits Core Ledger` is cryptographically authorized to invoke the `Payout Orchestrator`. All other services get a definitive 403 Forbidden.
> 2. **S3 Immutable Archiving (Repudiation):** Every outbound payload sent to the Central Bank is simultaneously synced to an AWS S3 `Settlement_Archive` bucket that has a strict **Object Lock (Compliance Mode)** enabled for 7 years. Not even the Root AWS Admin can delete the proof that we sent the file.
> 3. **Strict DynamoDB Idempotency (Tampering):** The Orchestrator leverages `attribute_not_exists` conditions on a `Payout_Job_Locks` table. The exact Payout UUID guarantees that if a network timeout occurs, we safely query the state machine instead of accidentally executing a double payment.

---

## 🔐 Team 4 (Platform): Auth-Service
**The Scenario:** The gatekeeper of the bank. Handles integrating with BankID (eID) and storing raw biometric or credential data for the users.

### 🔴 The Attack Vectors
| Threat Category | Specific Attack Vector | Severity |
| :--- | :--- | :--- |
| **Information Disclosure** | An engineer is debugging an API failure and looks at the Centralized Observability `OpenSearch` logs, explicitly viewing a plaintext customer National ID (PNR) or password in the JSON HTTP body dumps. | 🔴 CRITICAL |
| **Spoofing (Session Hijack)** | A customer's smartphone is stolen while unlocked. The thief uses the active OAuth Access Token to drain the bank accounts while the customer frantically calls support to freeze their account. | 🔴 CRITICAL |

### 🟢 The Architectural Defenses
> [!CAUTION]
> **Defense-in-Depth applied here:**
> 1. **Opaque Tokens & Stateful Validation (Spoofing):** The architecture mandates short-lived (15-minute) Opaque reference tokens rather than self-contained JWTs. When the customer calls support, the agent clicks "Revoke Sessions". The backend instantly deletes the token from DynamoDB. The thief's next API call hits API Gateway, validates the token against DynamoDB, discovers it is deleted, and instantly locks the app with a `401 Unauthorized`.
> 2. **Kinesis Firehose PII Masking (Information Disclosure):** Telemetry is not sent straight to OpenSearch. It passes through a Kinesis Firehose Lambda Transformation. This script actively searches for regex strings matching PNRs, emails, or Auth headers in the logs, and replaces them with `***[REDACTED]***` before the data even hits the searchable index. None of the backend engineers can view plaintext PII.
