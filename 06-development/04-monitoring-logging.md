# Incident Scenario: The Phantom Double-Tax (A Study in Distributed Tracing)
 
 **Date:** 2026-03-23
 **Severity:** SEV-1 (Financial Ledger Discrepancy)
 **Services Impacted:** Core Ledger (Team 2), Tax Engine (Team 3)
 
 ---
 
 ## 1. The Scenario & The Flow
 
 The system is running smoothly. A customer's `FIXED_TERM` deposit reaches maturity. The automated flow begins:
 
 1. **(Team 2) Midnight Yield Engine:** Calculates the interest earned, strictly commits the `IBR` (Interest Bearing) transaction to the Aurora ledger, and publishes an `AccountMatured` event to EventBridge.
 2. **(Team 3) Tax Engine:** An AWS Lambda triggered by the `AccountMatured` event. It calculates the mandatory 30% capital gains tax on the total yield.
 3. It synchronously calls the Core Ledger API: `POST /internal/v1/transactions` with `Type: TAX` to deduct the tax from the user's balance.
 4. **(Team 3) Payout Orchestrator:** Finally wires the remaining balance to the customer via SEPA.
 
 ### What Went Wrong?
 The Tax Engine successfully calculated the tax and successfully committed the `TAX` transaction to the Core Ledger API. 
 
 However, exactly 3 milliseconds *after* receiving the `201 Created` response from the Ledger, the Tax Engine Lambda experienced an unexpected **Out-Of-Memory (OOM) crash** due to a massive memory leak in a logging library. Because the Lambda crashed abruptly, it never sent a successful `ACK` (acknowledgment) back to AWS EventBridge.
 
 **The Retry Trap:** 
 EventBridge, assuming the payload wasn't processed, relied on its DLQ/Retry policy and fired the exact same `AccountMatured` event back to a fresh Tax Engine Lambda 5 minutes later. 
 
 The fresh Lambda calculated the tax again, generated a brand new random `Idempotency-Key` via `uuid.v4()`, and sent a *second* `TAX` deduction to the Core Ledger. Because the `Idempotency-Key` was different, the Core Ledger processed it as a completely new, valid transaction. **The customer was taxed 60% instead of 30%.**
 
 ---
 
 ## 2. How the Problem is Detected
 
 In a microservice universe, waiting for customers to complain about lost money is unacceptable. This discrepancy is spotted by two automated layers:
 
 1. **Proactive System Alerts (OpenSearch):** 
    - The Platform Team (Team 4) has a CloudWatch alarm looking for `"Lambda OutOfMemoryError"` in the Tax Engine logs. This fires a moderate PagerDuty alert to the on-call engineer warning of infrastructure instability.
 2. **Financial Reconciliation Cron (Midnight Checker):**
    - The ultimate safety net. A nightly CRON job aggregates all `IBR` yields generated that day and compares them to the total `TAX` deducted. 
    - It triggers a SEV-1 Slack Alert: `🚨 Anomaly Detected: Total TAX collected for batch 2026-03-23 exceeds the legal 30.00% threshold (Currently at exactly 60.00%). Ledger imbalance suspected.`
 
 ---
 
 ## 3. Root Cause Investigation (Distributed Tracing)
 
 The on-call engineer receives the alert. The customer hasn't even woken up yet. The engineer opens **Kibana (OpenSearch)** to trace the logs.
 
 1. **Querying the Anomaly:** The engineer searches the logs for today's transactions grouped by `customerId`. They use the query: `transactionType: "TAX"`.
 2. **Spotting the Duplicate:** They immediately spot a customer who had two identical `TAX` deductions (e.g., `-15.00 EUR`), timestamped exactly 5 minutes apart.
 3. **Tracing the Correlation ID:** They grab the `X-Correlation-ID` from both transactions to trace them backwards through the API Gateway.
    - Transaction 1: `Correlation-ID: a1b2c3d4`
    - Transaction 2: `Correlation-ID: x9y8z7w6`
 4. **Tracing the Event Trigger:** The engineer filters the logs further upstream by `service: "tax-engine"`. They realize that for *both* correlation IDs, the Lambda was triggered by the exact same EventBridge Event Payload: `event.id: "evt-999-matured"`.
 5. **Finding the Smoking Gun:** The engineer now looks at the HTTP request payload the Tax Engine sent to the Core Ledger. 
    - Request 1: `Idempotency-Key: 1111-2222`
    - Request 2: `Idempotency-Key: 8888-9999`
 
 **The Conclusion:** The engineer realizes the code generates a random UUID for the Idempotency-Key upon execution. When EventBridge retried the same event due to the OOM crash, a new Idempotency-Key was minted, bypassing the Ledger's idempotency protection database!
 
 ---
 
 ## 4. The Fix & Resolution
 
 ### Step A: Incident Mitigation (Data Fix)
 1. The engineer writes an auditable runbook script.
 2. They target the impacted `customerId` and execute a manual compensating transaction (`POST /internal/v1/transactions`) with `Type: PYI` (Pay In), effectively refunding the duplicated 15.00 EUR tax.
 3. The ledger is immediately re-balanced.
 
 ### Step B: Architectural Code Fix (Preventative)
 1. The engineer patches the Tax Engine code. Instead of using a random UUID for the Idempotency-Key, they switch to a **Deterministic Hash**.
 2. Code changes to: `Idempotency-Key = SHA256(customer.accountId + "TAX" + event.id)`
 3. If EventBridge ever retries that same `AccountMatured` event in the future, the exact same hash is generated. When the Tax Engine sends it to the Core Ledger, the Ledger's Aurora database will respond with `200 OK (Already Processed)`, preventing the double-charge natively!
 
 ### Step C: Post-Mortem
 1. The team conducts an Architecture Decision Record / Post-Incident meeting to enforce a new rule: **All event-driven consumers must generate Idempotency-Keys deterministically based on the incoming Event ID, never randomly.**
