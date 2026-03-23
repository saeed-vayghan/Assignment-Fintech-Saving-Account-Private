# Team 4: Platform & Shared — Data Models

#### **Primary Database:** Amazon DynamoDB (Auth) & ELK Stack (OpenSearch)

#### **Justification:**
- Platform manages active authentication states and massive telemetry/log aggregation across the entire bank.
- They do not store long-lived financial domain state.
- Custom rate-limiting datastores (like Redis) are explicitly avoided in favor of native AWS WAF and API Gateway Usage Plans.

---


## 1. Table: `Auth_Users` (DynamoDB)
The persistent, master record of identity mapping. This table never expires.

*   **Partition Key (`PK`):** `USER#<auth_identity_id>`
*   **Attributes:**
    *   `national_id_hash` (String - Cryptographically hashed Swedish PNR).
    *   `customer_id` (String - Optional until onboarding completes).
    *   `app_roles` (List of Strings - The master record of permissions: `[Unverified]`, `[Verified_Customer]`, `[Admin]`).
    *   `created_at` (String - ISO8601)
*   **GSI (Global Secondary Index):**
    *   `GSI1-PK`: `HASH#<national_id_hash>` -> Enables finding an existing identity instantly when a user logs in via BankID.

---

## 2. Table: `Auth_Tokens` (DynamoDB)
Tracks active, stateful opaque access tokens issued to users.

*   **Partition Key (`PK`):** `TOKEN#<hashed_access_token>`
*   **Attributes:**
    *   `auth_identity_id` (String - Map to user's identity)
    *   `customer_id` (String - Core banking ID injected by API Gateway)
    *   `app_roles` (List of Strings - `[Unverified]`, `[Verified_Customer]`).
    *   `expires_at` (Number - Epoch TTL for automatic cleanup)
*   **GSI (Global Secondary Index):**
    *   `GSI1-PK`: `USER#<auth_identity_id>` -> Allows finding/revoking all active tokens for a specific user.

---

## 3. Store: `Refresh_Tokens` (DynamoDB)
Allows the Platform team to instantly revoke access across all microservices if a user reports their device stolen.

*   **Partition Key (`PK`):** `TOKEN#<hashed_refresh_token>`
*   **Attributes:**
    *   `auth_identity_id` (String)
    *   `device_fingerprint` (String)
    *   `expires_at` (TTL Timestamp)
    *   `revoked` (Boolean)
*   **GSI (Global Secondary Index):**
    *   `GSI1-PK`: `USER#<auth_identity_id>` -> Allows a user to click "Logout everywhere" to instantly find and revoke all active tokens.

## 3. Index: `Alborz_Central_Telemetry` (OpenSearch / ELK)
The massive, searchable index of all logs collected from Onboarding, Deposits, and Payments via the Kinesis Firehose stream.

*   **Index Pattern:** `alborz-logs-yyyy.mm`
*   **Document Structure (JSON):**
    *   `@timestamp` (ISO8601)
    *   `level` (`INFO`, `WARN`, `ERROR`, `FATAL`)
    *   `environment` (`prod`, `staging`, `dev` - mandatory filter to prevent cross-tenant diagnostic bleed)
    *   `service_name` (e.g., `core-ledger-svc`, `interest-engine-lambda`)
    *   `trace_id` (String - OpenTelemetry UUID injected by the API Gateway to trace a single request across multiple microservices)
    *   `message` (String - Log payload)
    *   `customer_id` (String - ALWAYS explicitly mapped, allowing customer support to punch in an ID and instantly see exactly what failed).
    *   `pii_redacted` (Boolean - `true`. All raw SSNs or Emails are masked by Firehose before indexing into ELK).

## 4. Index: `Alborz_Compliance_Audit` (OpenSearch / ELK)
A strictly isolated, immutable index exclusively dedicated to compliance actions (PEP screening results, KYC submissions, account state changes). This fulfills the strict regulatory audit requirements without getting lost in noisy operational telemetry logs.

*   **Index Pattern:** `alborz-audit-yyyy.mm` (or managed via OpenSearch Index State Management (ISM) Rollover aliases to split partitions at 30GB regardless of date).
*   **Document Structure (JSON):**
    *   `@timestamp` (ISO8601)
    *   `actor_type` (`CUSTOMER`, `EMPLOYEE`, `SYSTEM`)
    *   `actor_id` (String - e.g., the UUID of the Customer, the Okta/SSO ID of the Compliance Officer, or `Underwriting_Engine`)
    *   `customer_id` (String)
    *   `action_type` (`KYC_SUBMITTED`, `PEP_SCREENING_MATCH`, `ACCOUNT_STATUS_CHANGED`)
    *   `old_value` / `new_value` (JSON - for state transitions)
    *   `reasoning` (String - officer notes for overrides)

## 5. Table: `Notification_Log` (DynamoDB)
Maintains an immutable record of every outbound SMS and Email sent to customers to guarantee communication compliance and prevent duplicate alerts.

*   **Partition Key (`PK`):** `CUSTOMER#<customer_id>`
*   **Sort Key (`SK`):** `NOTIFY#<timestamp>#<message_id>`
*   **Attributes:**
    *   `channel` (String - `SMS`, `EMAIL`)
    *   `template_id` (String - Links to the localized template used)
    *   `status` (String - `DELIVERED`, `BOUNCED`, `FAILED_DLQ`)
    *   `idempotency_key` (String - The ID of the triggering Pub/Sub event)
    *   `failed_status` (String, Nullable - Populated only if SES/SNS delivery fails)
*   **GSI (Global Secondary Index):**
    *   `GSI1-PK`: `FAILED_STATUS#<status>` -> **Sparse Index**. Only physicalizes failed delivery attempts, allowing Support queries without index throttling.
