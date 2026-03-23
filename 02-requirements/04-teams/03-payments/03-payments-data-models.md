# Team 3: Payments & Accounting — Data Models

#### **Primary Database:** Amazon DynamoDB (State Store) & Amazon S3 (File Archive)

#### **Justification:**
- The core financial math is committed directly to the Deposits ledger (Aurora).
- Payments only needs a database to guarantee batch job state tracking, preventing the catastrophic "Double Payout" bug.
- S3 provides infinitely scalable, cheap storage for massive banking XML files.

---

## 1. Table: `Payout_Job_Locks` (DynamoDB)
Maintains the exact state of money currently moving out of the bank.

*   **Partition Key (`PK`):** `PAYOUT_JOB#<uuid>`
*   **Attributes:**
    *   `ledger_event_id` (String - the exact withdrawal event from Deposits)
    *   `account_id` (String)
    *   `amount` (Number)
    *   `destination_iban` (String)
    *   `status` (String - `QUEUED`, `PROCESSING`, `CLEARED_SUCCESS`)
    *   `failed_status` (String, Nullable - `FAILED_RETRY`, `FAILED_DLQ`. Only populated if the job fails)
    *   `created_at`, `updated_at` (Timestamps)
    *   `ttl` (Number - auto-purges old lock records after 30 days)
*   **GSI (Global Secondary Index):**
    *   `GSI1-PK`: `FAILED_STATUS#<status>` -> Implements a **Sparse Index**. Because 99.9% of payouts succeed, this GSI remains completely empty. It only indexes the 0.1% of failed jobs, allowing FinOps to query the DLQ instantly without experiencing "Hot Partition" throttling.
*   **Idempotency Strategy:**
    *   The Payout orchestrator uses an `attribute_not_exists(PK)` condition when creating the record. If the lambda retries, the DB write rejects, guaranteeing we do not send the money twice.

## 2. Table: `Inbound_CAMT_Message_Locks` (DynamoDB)
Maintains strict idempotency on incoming files so we never process the exact same payment twice.

*   **Partition Key (`PK`):** `CAMT_MESSAGE#<message_id>`
*   **Attributes:**
    *   `file_name` (String)
    *   `payment_reference` (String - maps back to Deposits account target)
    *   `amount` (Number)
    *   `status` (String - `PROCESSING`, `SUCCESS`)
    *   `failed_status` (String, Nullable - `FAILED_DLQ`. Only populated if XML parsing fails)
    *   `ttl` (Number - auto-purges after 30 days)
*   **GSI (Global Secondary Index):**
    *   `GSI1-PK`: `FAILED_STATUS#<status>` -> **Sparse Index**. Only indexes corrupted CAMT files for monitoring dashboards.

## 3. Bucket: `Alborz_Settlement_Archive` (Amazon S3)
Stores raw, immutable financial files transferred via SFTP.

*   **Path Structure:** `s3://alborz-settlement-archive/camt053/incoming/YYYY/MM/DD/`
*   **Object Properties:**
    *   `Key`: `camt053_bankofsweden_20261015_0900.xml`
    *   `Metadata (S3 Tags)`:
        *   `processed_status`: `SUCCESS` | `FAILED_VALIDATION`
        *   `rows_parsed`: `4052`
        *   `ingestion_job_id`: `<uuid>`
*   **Lifecycle Policies (Regulatory Requirement):**
    *   Transition to **S3 Standard-Infrequent Access (IA)** after 30 days.
    *   Transition to **S3 Glacier Flexible Retrieval** after 90 days for long-term immutable retention (7-10 years).

## 4. Table: `Tax_Withholding_Log` (DynamoDB)
A secondary ledger purely for regulatory reporting (M30). Tracks money the bank has withheld and owes to the government.

*   **Partition Key (`PK`):** `TAX_YEAR#<YYYY>`
*   **Sort Key (`SK`):** `CUSTOMER#<account_id>#EVENT#<ledger_event_id>` (Guarantees ultimate idempotency if the midnight cron job retries)
*   **Attributes:**
    *   `customer_tax_id` (String)
    *   `interest_earned` (Number)
    *   `tax_withheld` (Number)
    *   `tax_rate_applied` (Number - e.g. 0.30 for 30%)
    *   `pending_remittance_date` (ISO8601 Timestamp, Nullable - Only exists until the tax is formally paid to the government, then the attribute is deleted)
*   **GSI (Global Secondary Index):**
    *   `GSI1-PK`: `PENDING_REMITTANCE#<date>` -> **Sparse Index**. The absolute second the tax is paid via SEPA wire, the backend deletes the `pending_remittance_date` attribute. The row vanishes from the GSI instantly. This acts as an ultra-fast, perfectly distributed "To-Do List" of unpaid taxes and destroys the boolean Hot Partition.

## 5. Table: `Regulatory_Report_Tasks` (DynamoDB)
Maintains the idempotency and execution state of the monthly central bank reporting jobs (CRON).

*   **Partition Key (`PK`):** `REPORT_JOB#<YYYY-MM>`
*   **Attributes:**
    *   `status` (String - `PROCESSING`, `UPLOADED_TO_S3`, `TRANSMITTED_TO_CB`)
    *   `s3_archive_key` (String)
    *   `transmission_receipt` (String)
*   **GSI (Global Secondary Index):**
    *   `GSI1-PK`: `STATUS#FAILED` -> Sparse Index for diagnosing failed generation tasks.

## 6. Bucket: `Alborz_Regulatory_Archive` (Amazon S3)
Stores the generated XML transaction reports required by the central bank for 10-year immutable audit retention.
