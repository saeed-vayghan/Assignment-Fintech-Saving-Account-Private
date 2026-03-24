# Alborz Bank — Database Selection Guide


## 1. Onboarding Service

*Handles customer registration, KYC questionnaires, and ID state machines.*

* **Primary Database:** Amazon DynamoDB
* **Why we chose it:** 
  * **Flexible Schema:** Onboarding involves a lot of JSON documents (KYC answers, webhook payloads from Onfido/Plaid) which change often. NoSQL is perfect for this.
  * **Speed & Scale:** DynamoDB handles massive spikes in new user registrations effortlessly without managing servers.
  * **Write Sharding:** By sharding the Global Secondary Index on the application status (e.g., `STATUS#ACTIVE#SHARD#4`), Compliance Officers can query millions of applications with theoretically infinite read throughput, completely avoiding Hot Partition anti-patterns.
  * **Key-Value State:** Tracking a customer's state (`DRAFT` → `VERIFIED` → `ACTIVE`) is a simple, ultra-fast key lookup.
* **Secondary Option:** Amazon DocumentDB (MongoDB compatible). It is great for JSON, but it requires managing instances/clusters, which breaks our strict serverless-first rule.

---

## 2. Deposits & Transactions Service

*Handles core bank accounts, balances, and the financial ledger.*

* **Primary Database:** Amazon Aurora (PostgreSQL) - Serverless
* **Why we chose it:** 
  * **ACID Compliance is Mandatory:** Financial ledgers are the heart of the bank. We absolutely require strong relational logic, foreign keys, and transaction rollbacks to guarantee money is never lost, duplicated, or corrupted.
  * **Mathematical Rigor:** PostgreSQL has excellent, strict data types for financial math (avoiding rounding errors).
  * **Relational Data:** Customers, Accounts, and Transactions are highly relational and need to be joined together frequently.
  * **Strict Foreign Keys:** By anchoring accounts and dynamic interest rates to a relational `Product_Catalog` table, we categorically prevent corrupt data vectors (e.g., opening an `OVERNIGHT` account using a `FIXED_TERM` product code).
* **Secondary Option:** Amazon RDS for MySQL. It is also ACID compliant and battle-tested, but Postgres offers slightly better JSON support and stricter data typing constraints.

---

## 3. Payments & Accounting Service

*Handles massive End-Of-Month (EOM) batch jobs, SFTP XML parsing, and idempotency.*

* **Primary Database:** Amazon DynamoDB (State & Locks) + Amazon S3 (Files)
* **Why we chose it:** 
  * *(Note: Actual final transaction entries are saved to the Deposits Aurora DB. This service just calculates and processes them).*
  * **Idempotency Locks:** When processing thousands of payouts, DynamoDB's conditional writes are the perfect way to lock a job and guarantee we never process the exact same payout twice.
  * **File Storage:** S3 is the cheapest and most secure way to store the large XML (CAMT.053) files delivered by partner banks.
* **Secondary Option:** Amazon ElastiCache (Redis) for idempotency locks. Redis is incredibly fast for locking, but because it stores data in memory, it introduces a slight risk of losing lock data during a reboot compared to DynamoDB.

---

## 4. Auth-Service

*Handles user credentials, sessions, and authentication tokens.*

* **Primary Database:** Amazon DynamoDB
* **Why we chose it:** 
  * **Ultra-Low Latency:** Authentication APIs are called constantly. DynamoDB provides single-digit millisecond response times for fetching a user's password hash by their email address.
  * **No Connection Pooling Issues:** Unlike relational databases, DynamoDB communicates over HTTP. During a massive spike in logins, it won't crash due to "too many open connections."
* **Secondary Option:** Amazon Aurora (PostgreSQL). Good for keeping users relational, but connection limits can become a severe bottleneck during high traffic spikes.

---

## 5. Audit & Logging

*Handles compliance tracking and system observability.*

* **Primary Database:** ELK Stack (Elasticsearch / OpenSearch)
* **Why we chose it:** 
  * **Full-Text Search:** Compliance officers and developers need to quickly search millions of logs for a specific `correlationId` or `customerId`. 
  * **Indexing:** It automatically indexes every field in our structured logs, making dashboards and troubleshooting extremely fast.
* **Secondary Option:** CloudWatch Logs + Custom Node.js search scripts. Cheaper, but searching across massive logs textually using scripts is very slow and hard to visualize.

---
<br>

# A note on Distributed Consistency Models

Selecting the correct consistency model is the difference between a high-performance system and a system with "phantom" balances or double-spending exploits.

* **Strong Consistency:** Guarantees that every node in the cluster sees the absolute latest write immediately, preventing any stale or intermediate reads.
* **Sequential Consistency:** Ensures all processes see all operations in the same specific order, as if they were executed on a single processor.
* **Eventual Consistency:** Guarantees that if no new updates are made, all reads across all nodes will eventually return the same final value.
* **Causal Consistency:** Ensures that operations related by cause and effect are observed in the same specific order by all nodes across the system.

### Why and Where it matters for Alborz Bank

In this distributed architecture, selecting the wrong model can lead to catastrophic financial discrepancies or security breaches:

* **Strong Consistency is Mandatory:** The **Deposits & Ledger Service (Team 2)** must use Strong Consistency. If a customer spends £100, the system *must* coordinate across all nodes to ensure those funds are immediately and globally subtracted before another transaction attempt.
* **Eventual Consistency is Acceptable:** The **Notification Engine** and **Audit Analytics** can tolerate "stale" data for a few milliseconds, prioritizing high availability over instantaneous global synchronization.

