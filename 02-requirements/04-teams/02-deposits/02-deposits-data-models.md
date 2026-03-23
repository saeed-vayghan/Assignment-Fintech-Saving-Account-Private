# Team 2: Deposits & Transactions — Data Models

#### **Primary Database:** Amazon Aurora (PostgreSQL) Serverless

#### **Justification:**
- Complete ACID compliance for financial integrity.
- Relational strictness prevents data corruption.
- Event Sourcing is used for immutable auditability.

---

## 1. Table: `Accounts`
Stores the high-level configuration and lifecycle state of a savings account.

*   **Columns:**
    *   `account_id` (UUID, Primary Key)
    *   `account_number_iban` (VARCHAR, Unique - critical for matching incoming CAMT PYI payments)
    *   `customer_id` (UUID, Foreign Key mapping back to Onboarding)
    *   `account_type` (VARCHAR - `OVERNIGHT`, `FIXED_TERM`)
    *   `currency` (VARCHAR - `EUR`, `SEK`)
    *   `status` (VARCHAR - `PENDING_FUNDING`, `ACTIVE`, `FROZEN`, `MATURED`, `CLOSED`)
    *   `product_code` (VARCHAR, Foreign Key mapping to `Product_Catalog` - Used to fetch yield rates and lock constraints)
    *   `locked_interest_rate_pct` (DECIMAL, Nullable - Only populated for `FIXED_TERM` accounts to legally lock the rate at creation)
    *   `term_months` (INTEGER, Nullable - e.g., 3, 6, or 12)
    *   `maturity_date` (TIMESTAMP, Nullable - STRICTLY NULL until the first PYI event transitions the account from `PENDING_FUNDING` to `ACTIVE`)
    *   `auto_rollover` (BOOLEAN DEFAULT FALSE - automatically renew at maturity date)
    *   `created_at` (TIMESTAMP)
    *   `updated_at` (TIMESTAMP)
    *   `closed_at` (TIMESTAMP DEFAULT NULL)
*   **Indexes:**
    *   `UNIQUE INDEX idx_account_number_iban` -> High-speed O(1) matching for incoming CAMT payments.
    *   `INDEX idx_customer_id` -> Instantly look up all accounts owned by one customer.
    *   `INDEX idx_status_maturity` -> Crucial for the midnight Account Maturity Cron Job.

## 2. Table: `Ledger_Events` (The Write Model / Event Store)
Append-only log of every single financial mutation. Rows are never deleted or updated.

*   **Columns:**
    *   `event_id` (UUID, Primary Key)
    *   `account_id` (UUID, Foreign Key, Indexed)
    *   `sequence_number` (INTEGER, Unique Constraint mixed with `account_id` - enforcing strict chronological, optimistic locking)
    *   `idempotency_key` (VARCHAR, Unique Constraint - prevents double processing)
    *   `transaction_type` (VARCHAR - `DEPOSIT_PYI`, `WITHDRAWAL_PYO`, `INTEREST_IBR`, `TAX`, `CANCELLATION_CAN`)
    *   `direction` (VARCHAR - `CREDIT`, `DEBIT`)
    *   `currency` (VARCHAR - strictly enforces the amount matches the `Account.currency` at DB level)
    *   `amount` (DECIMAL(19,4) - strictly positive absolute value)
    *   `balance_after_event` (DECIMAL(19,4) - running total check)
    *   `transaction_date` (TIMESTAMP)
    *   `settlement_date` (TIMESTAMP)
    *   `metadata` (JSONB - e.g., CAMT reference ID, Payment instruction ID)
*   **Indexes:**
    *   `UNIQUE INDEX idx_account_id_seq` -> `(account_id, sequence_number)`
    *   `UNIQUE INDEX idx_idempotency_key` -> Absolute database-level lock preventing double-withdrawal bugs.
    *   `INDEX idx_account_id_date` -> Instantly fetch transaction history for the user dashboard.

> **Architectural Note: Defeating Ledger Race Conditions (Optimistic Locking)**
> In a highly concurrent financial system, two withdrawals acting on the identical `$100` balance at the exact same millisecond could result in the total balance incorrectly settling at `$50` rather than `$0`. 
> 
> By forcing the API to append events using a strictly incrementing `sequence_number` combined with a central `UNIQUE CONSTRAINT (account_id, sequence_number)`, the database operates as a high-speed physics engine. The first withdrawal claims `sequence_number = 14`. The second withdrawal attempts to claim `sequence_number = 14`, crashes into the SQL unique constraint violation, mathematically rolls back its parent transaction, fetches the actual new `$50` balance, and re-calculates safety margins as `sequence_number = 15`. This enforces absolute ACID financial consistency.

## 3. View / Materialized Table: `Customer_Balance` (The CQRS Read Model)
A highly optimized, denormalized read model populated asynchronously by the Projection Service.

*   **Columns:**
    *   `account_id` (UUID, Primary Key)
    *   `customer_id` (UUID, Indexed)
    *   `current_balance` (DECIMAL(19,4))
    *   `available_balance` (DECIMAL(19,4) - minus pending holds)
    *   `accrued_interest_mtd` (DECIMAL(19,4) - tracks daily calculated interest before it is formally paid via IBR)
    *   `last_event_sequence` (INTEGER - tracks the highest `sequence_number` successfully projected from the CDC stream to guarantee chronologically perfect read-model idempotency)
    *   `last_calculated_at` (TIMESTAMP)

## 4. Table: `Interest_Rates`
Tracks historical and upcoming product rates with effective dates.

*   **Columns:**
    *   `rate_id` (UUID, Primary Key)
    *   `product_code` (VARCHAR, Foreign Key mapping to `Product_Catalog`)
    *   `annual_percentage_yield` (DECIMAL(5,4) - e.g., 0.0450 for 4.5%)
    *   `effective_from_date` (TIMESTAMP)
    *   `effective_to_date` (TIMESTAMP, Nullable - active until changed)
    *   `created_by_admin_id` (VARCHAR)
    *   `created_at` (TIMESTAMP)

## 5. Table: `Product_Catalog`
Master dictionary of all savings products offered by the bank. Anchors the `product_code` constraints to prevent impossible account configurations (e.g., trying to open an Overnight account with a Fixed-Term product code).

*   **Columns:**
    *   `product_code` (VARCHAR, Primary Key - e.g., `OVR_SAVINGS_V1`)
    *   `product_name` (VARCHAR)
    *   `account_type` (VARCHAR - `OVERNIGHT`, `FIXED_TERM`)
    *   `currency` (VARCHAR - `EUR`, `SEK`)
    *   `max_balance_limit` (DECIMAL(19,4), Nullable)
    *   `is_active` (BOOLEAN DEFAULT TRUE)
    *   `created_at` (TIMESTAMP)
