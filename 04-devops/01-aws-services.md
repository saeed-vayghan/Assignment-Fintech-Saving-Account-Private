# AWS Services: Where & Why

This document outlines the justification and placement of core AWS services within the Alborz Bank Savings Account project architecture.

## Networking & Access Control

### 1. AWS API Gateway
- **Where:** Front door for all microservices (Onboarding, Deposits, Payments, Platform).
- **Why:** Centralizes routing, TLS termination, and standardizes authentication (via custom authorizers). Enables rate limiting and Web Application Firewall (WAF) integration against DoS attacks.

### 2. IAM (Identity and Access Management)
- **Where:** Globally across all AWS resources, especially for service-to-service communication.
- **Why:** Enforces the Principle of Least Privilege. We use IAM execution roles instead of hardcoded passwords so the `Orchestrator` can invoke `Deposits API` cryptographically, rejecting spoofed internal calls.

### 3. Security Groups
- **Where:** Attached to Aurora RDS, ECS/Fargate containers, and Lambda VPC endpoints.
- **Why:** Provides foundational network-level isolation. E.g., Aurora PostgreSQL only accepts traffic exclusively from the `Deposits` and `Payments` security groups.

## Compute & Scaling

### 4. Auto Scaling (ECS/EC2)
- **Where:** Fargate/ECS clusters hosting long-running services (BFFs, Core Ledger APIs).
- **Why:** Handles predictable, horizontal scaling of the synchronous path based on CPU and memory metrics to maintain `< 200ms` API response SLAs during heavy banking hours.

### 5. Lambda Auto Scaling (Concurrency)
- **Where:** Event-driven jobs (Inbound CAMT Parsers, Onboarding Webhook Handlers).
- **Why:** Scales instantaneously from 0 to 1,000 for unpredictable bursts of traffic (like a massive batch of webhook payloads), while using mathematically constrained concurrency limits connected to SQS to avoid database exhaustion.

## Messaging

### 6. Amazon SQS (Simple Queue Service)
- **Where:** Between external Webhooks and our parsers, and between EventBus endpoints.
- **Why:** Acts as a FIFO buffer zone. Absorbs huge traffic spikes safely so downstream databases aren't overwhelmed (e.g., IDV Webhook Handler).

### 7. Amazon SNS (Simple Notification Service)
- **Where:** Integrated with the Platform team's Notification Engine.
- **Why:** Fanout messaging. It triggers parallel actions (e.g., SMS to customer, Email to customer, Notification to Admin Portal) when an account is matured or a deposit is successfully settled.

## Storage
 
### 8. Amazon S3
- **Where:** CAMT Archiving, Regulatory Reports, Legal Assets.
- **Why:** Immutable, highly-durable blob storage. We enforce **S3 Object Lock (Compliance Mode)** on the Central Bank settlement files to guarantee non-repudiation and auditability for 7 years.
 
## Databases

### 9. Amazon Aurora PostgreSQL
- **Where:** Core Ledger (Deposits Team).
- **Why:** Delivers strict ACID compliance, relational integrity, and pessimistic locking capabilities required for managing critical financial balances and double-entry ledgers without risk of race conditions.

### 10. Amazon DynamoDB
- **Where:** Payout Job Locks, Onboarding state, Opaque Token stores.
- **Why:** Provides single-digit millisecond NoSQL performance for high-concurrency event-driven architectures. Perfect for idempotency locks (Payments) and managing fast-expiring opaque tokens (Platform).

## Security

### 11. AWS KMS (Key Management Service)
- **Where:** Aurora DBs, S3 Buckets, and AWS Secrets Manager.
- **Why:** Enforces encryption-at-rest. The central key is required for decoding PNRs and sensitive financial logs. Even if a snapshot is stolen, the data is unreadable without KMS execution rights.

## Observability
 
### 12. Amazon CloudWatch
- **Where:** Attached to all APIs, Lambdas, and Databases.
- **Why:** Centralizes Application/Infrastructure metrics. Defines CloudWatch Alarms (e.g., Error Rates > 1%) to notify the Platform on-call teams automatically.
