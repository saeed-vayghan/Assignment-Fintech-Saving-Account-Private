# Alborz Bank — Savings Account: Requirements Checklist

This checklist maps every requirement to its delivery status, priority, team ownership, phase, and MoSCoW category. See teams.md for team definitions and roadmap.md for phased delivery plan.

**Legend:**
- **Status:** `Not Started` | `In Progress` | `Done`
- **Covered:** ✅ = tracked in roadmap | ⚠️ = partially covered | ❌ = was missing (now added)
- **MoSCoW:** M = Must Have | S = Should Have | C = Could Have | W = Won't Have

---

## 1. Customer Eligibility

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 1.1 | Only individual (private) persons can open accounts | ✅ | M | Not Started | Onboarding | 1 | M25 |
| 1.2 | Customer must be a resident of the bank's operating country | ✅ | M | Not Started | Onboarding | 1 | M25 |
| 1.3 | Minors (under 18) are not eligible | ✅ | M | Not Started | Onboarding | 1 | M25 |
| 1.4 | Trusts, businesses, governmental entities are not eligible | ✅ | M | Not Started | Onboarding | 1 | M25 |

---

## 2. Deposit Types

### 2.1 Fixed Term Deposit

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 2.1.1 | Create fixed term deposit (3, 6, or 12 months) | ✅ | M | Not Started | Deposits | 1 | M5 |
| 2.1.2 | Fixed interest rate (does not change during term) | ✅ | M | Not Started | Deposits | 1 | M5 |
| 2.1.3 | Rollover at maturity (automatic renewal) | ✅ | M | Not Started | Deposits | 1 | M5 |
| 2.1.4 | No partial withdrawals during term | ✅ | M | Not Started | Deposits | 1 | M5 |
| 2.1.5 | Early cancellation follows CAN transaction flow | ✅ | M | Not Started | Deposits | 2 | M7 |
| 2.1.6 | Early cancellation may involve penalty or reduced interest | ✅ | M | Not Started | Deposits | 2 | M7 |

### 2.2 Overnight Savings Account

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 2.2.1 | Create overnight savings account (no lock-in) | ✅ | M | Not Started | Deposits | 1 | M6 |
| 2.2.2 | Variable interest rate (can change over time) | ✅ | M | Not Started | Deposits | 1 | M6 |
| 2.2.3 | Interest calculated daily based on balance | ✅ | M | Not Started | Payments | 2 | M9 |
| 2.2.4 | Customer can top up (add money) at any time | ✅ | M | Not Started | Deposits | 1 | M6 |
| 2.2.5 | Customer can partially withdraw at any time | ✅ | M | Not Started | Deposits | 1 | M6 |

---

## 3. Transaction Types

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 3.1 | PYI — Pay-In: receive money from customer's external account | ✅ | M | Not Started | Deposits | 2 | M7 |
| 3.2 | PYI — covers initial deposit and later top-ups (overnight) | ✅ | M | Not Started | Deposits | 2 | M7 |
| 3.3 | PYO — Pay-Out: send money back on maturity or partial withdrawal | ✅ | M | Not Started | Deposits | 2 | M7 |
| 3.4 | PYO — only to verified transaction account | ✅ | M | Not Started | Deposits | 2 | M7 |
| 3.5 | CAN — Cancellation: early termination of fixed term deposit | ✅ | M | Not Started | Deposits | 2 | M7 |
| 3.6 | CAN — may involve penalty or reduced interest | ✅ | M | Not Started | Deposits | 2 | M7 |
| 3.7 | IBR — Interest Booking: ledger-only, no real money movement | ✅ | M | Not Started | Payments | 2 | M7 |
| 3.8 | TAX — Tax Deduction: withholding tax, reduces payout amount | ✅ | M | Not Started | Payments | 2 | M7 |

---

## 4. Payment Import

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 4.1 | Receive payments in market default currency | ✅ | M | Not Started | Payments | 2 | M8 |
| 4.2 | Payments exported as CAMT files (ISO 20022 XML) | ✅ | M | Not Started | Payments | 2 | M8 |
| 4.3 | CAMT files placed on SFTP server | ✅ | M | Not Started | Payments | 2 | M8 |
| 4.4 | System polls/fetches CAMT files from SFTP | ✅ | M | Not Started | Payments | 2 | M8 |
| 4.5 | Parse each CAMT file | ✅ | M | Not Started | Payments | 2 | M8 |
| 4.6 | Match payment to customer account using payment reference | ✅ | M | Not Started | Payments | 2 | M8 |
| 4.7 | Create corresponding PYI transaction | ✅ | M | Not Started | Payments | 2 | M8 |

---

## 5. Accounting Jobs (Background Processes)

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 5.1 | Payout job: matured fixed term → PYO transaction | ✅ | M | Not Started | Payments | 2 | M11 |
| 5.2 | Payout job: non-renewed fixed term → PYO transaction | ✅ | M | Not Started | Payments | 2 | M11 |
| 5.3 | Payout job: early cancellation → CAN transaction | ✅ | M | Not Started | Payments | 2 | M11 |
| 5.4 | Payout job: overnight withdrawal request → PYO transaction | ✅ | M | Not Started | Payments | 2 | M11 |
| 5.5 | Interest calculation: fixed rate for term deposits | ✅ | M | Not Started | Payments | 2 | M9 |
| 5.6 | Interest calculation: variable/daily rate for overnight | ✅ | M | Not Started | Payments | 2 | M9 |
| 5.7 | Interest calculation: store as IBR transaction | ✅ | M | Not Started | Payments | 2 | M9 |
| 5.8 | Tax calculation: compute withholding tax on accrued interest | ✅ | M | Not Started | Payments | 2 | M10 |
| 5.9 | Tax calculation: store as TAX transaction | ✅ | M | Not Started | Payments | 2 | M10 |

---

## 6. Legal & Compliance — Identity Verification

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 6.1 | Collect: full legal name | ✅ | M | Not Started | Onboarding | 1 | M1 |
| 6.2 | Collect: date of birth | ✅ | M | Not Started | Onboarding | 1 | M1 |
| 6.3 | Collect: nationality | ✅ | M | Not Started | Onboarding | 1 | M1 |
| 6.4 | Collect: national ID or passport number | ✅ | M | Not Started | Onboarding | 1 | M1 |
| 6.5 | Collect: proof of address | ✅ | M | Not Started | Onboarding | 1 | M1 |
| 6.6 | Validate document format and consistency | ✅ | M | Not Started | Onboarding | 1 | M1 |
| 6.7 | Optional: 3rd-party identity verification (Onfido/IDnow) | ✅ | M | Not Started | Onboarding | 1 | M1 |
| 6.8 | Store verification status (VERIFIED/PENDING/REJECTED) | ✅ | M | Not Started | Onboarding | 1 | M1, M28 |
| 6.9 | Blocking: cannot proceed until verified | ✅ | M | Not Started | Onboarding | 1 | M28 |

---

## 7. Legal & Compliance — Bank Account Validation

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 7.1 | IBAN format validation (structure + check digits) | ✅ | M | Not Started | Onboarding | 1 | M2 |
| 7.2 | Ownership verification (micro-deposit, name-match API, or Plaid/Tink) | ✅ | M | Not Started | Onboarding | 1 | M2 |
| 7.3 | Mark account as VERIFIED → stored as verified transaction account | ✅ | M | Not Started | Onboarding | 1 | M2 |
| 7.4 | All future payouts only to this verified account | ✅ | M | Not Started | Onboarding | 1 | M2 |
| 7.5 | Blocking: cannot activate until bank account is verified | ✅ | M | Not Started | Onboarding | 1 | M28 |

---

## 8. Legal & Compliance — PEP/Sanctions Screening

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 8.1 | Send name, DOB, nationality to screening provider | ✅ | M | Not Started | Onboarding | 1 | M3 |
| 8.2 | Handle result: NO_MATCH, POTENTIAL_MATCH, CONFIRMED_MATCH | ✅ | M | Not Started | Onboarding | 1 | M3 |
| 8.3 | If match: flag customer record with WARNING status | ✅ | M | Not Started | Onboarding | 1 | M3 |
| 8.4 | If match: create compliance case for manual review | ✅ | M | Not Started | Onboarding | 2 | M3, S2 |
| 8.5 | Compliance officer can approve or reject | ✅ | S | Not Started | Onboarding | 2 | S2 |
| 8.6 | Non-blocking, but pauses activation until resolved | ✅ | M | Not Started | Onboarding | 1 | M3, M28 |

---

## 9. Legal & Compliance — KYC Questionnaire

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 9.1 | 5 questions (configurable by compliance/product team) | ✅ | M | Not Started | Onboarding | 2 | M4 |
| 9.2 | Question types: single-select | ✅ | M | Not Started | Onboarding | 2 | M4 |
| 9.3 | Question types: multi-select | ✅ | M | Not Started | Onboarding | 2 | M4 |
| 9.4 | Question types: free text | ✅ | M | Not Started | Onboarding | 2 | M4 |
| 9.5 | Questions and options are admin-configurable (not hardcoded) | ✅ | M | Not Started | Onboarding | 2 | M4 |
| 9.6 | Answers stored for audit and regulatory reporting | ✅ | M | Not Started | Onboarding | 2 | M4 |
| 9.7 | Blocking: must be completed before account activation | ✅ | M | Not Started | Onboarding | 2 | M28 |

---

## 10. Technical Deliverables — Architecture Diagrams

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 10.1 | Onboarding sequence diagram (full customer journey) | ✅ | M | Not Started | Architect | 1 | M29 |
| 10.2 | Onboarding: show all 4 compliance steps + external services | ✅ | M | Not Started | Architect | 1 | M29 |
| 10.3 | Onboarding: decision points (failure, PEP match, abandon) | ✅ | M | Not Started | Architect | 1 | M29 |
| 10.4 | Onboarding: API calls between frontend, backend, 3rd parties | ✅ | M | Not Started | Architect | 1 | M29 |
| 10.5 | Onboarding: customer state transitions | ✅ | M | Not Started | Architect | 1 | M29 |
| 10.6 | Customer web app: view deposit accounts (FTD + overnight) | ✅ | M | Not Started | Architect | 2 | M29 |
| 10.7 | Customer web app: withdrawal request flow | ✅ | S | Not Started | Deposits | 2 | S1 |
| 10.8 | Customer web app: transaction history (PYI, PYO, CAN, IBR, TAX) | ✅ | M | Not Started | Deposits | 2 | M12 |
| 10.9 | Customer web app: frontend-backend communication (REST/GraphQL) | ✅ | M | Not Started | Deposits | 2 | M12 |
| 10.10 | Customer web app: real-time/near-real-time balance + interest | ✅ | M | Not Started | Deposits | 2 | M12 |

---

## 11. Best Practices — Security

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 11.1 | Authentication & authorization (Auth-Service) | ✅ | M | Not Started | Platform | 1 | M13 |
| 11.2 | Encryption at rest (AWS KMS) | ✅ | M | Not Started | Platform | 1 | M14 |
| 11.3 | Encryption in transit (TLS) | ✅ | M | Not Started | Platform | 1 | M14 |
| 11.4 | Secrets management (Secrets Manager, SSM) | ✅ | M | Not Started | Platform | 1 | M26 |
| 11.5 | API security: rate limiting | ✅ | M | Not Started | Platform | 2 | M21 |
| 11.6 | API security: input validation | ✅ | M | Not Started | Platform | 2 | M21 |
| 11.7 | API security: CORS policies | ✅ | M | Not Started | Platform | 2 | M21 |
| 11.8 | PII handling and data masking in logs | ✅ | M | Not Started | Platform | 1 | M24 |

---

## 12. Best Practices — Monitoring

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 12.1 | Application-level metrics (latency, error rates, throughput) | ✅ | M | Not Started | Platform | 1 | M18 |
| 12.2 | Business-level metrics (accounts opened, deposits, flags) | ✅ | M | Not Started | Platform | 1 | M18 |
| 12.3 | Infrastructure monitoring (CPU, memory, Lambda, API Gateway) | ✅ | M | Not Started | Platform | 1 | M18 |
| 12.4 | Alerting strategy (triggers, severity, escalation) | ✅ | M | Not Started | Platform | 1 | M20 |
| 12.5 | Tools: CloudWatch, CloudTrail, AWS Config, Grafana | ✅ | M | Not Started | Platform | 1 | M18 |

---

## 13. Best Practices — Logging

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 13.1 | Structured logging (JSON) with correlation IDs | ✅ | M | Not Started | Platform | 1 | M23 |
| 13.2 | Log levels: INFO, WARN, ERROR | ✅ | M | Not Started | Platform | 1 | M23 |
| 13.3 | Centralized log aggregation (CloudWatch Logs, ELK Stack) | ✅ | M | Not Started | Platform | 1 | M19 |
| 13.4 | Audit logging for compliance actions (immutable) | ✅ | M | Not Started | Platform | 1 | M15 |
| 13.5 | PII redaction (national ID, IBAN masked) | ✅ | M | Not Started | Platform | 1 | M24 |

---

## 14. Best Practices — Operational Excellence

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 14.1 | Runbooks for common operational tasks | ✅ | C | Not Started | Platform | 4 | C7 |
| 14.4 | Operational readiness reviews (ORR) before releases | ✅ | S | Not Started | Platform | 4 | Phase 4 |
| 14.5 | Change management: review & approval for prod changes | ✅ | S | Not Started | Platform | 3 | Phase 3 |

---

## 15. Best Practices — Developer Experience

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 15.1 | Infrastructure as Code (AWS CDK / CloudFormation) | ✅ | M | Not Started | Platform | 1 | M16 |
| 15.2 | CI/CD pipeline (build, test, deploy, quality gates) | ✅ | M | Not Started | Platform | 1 | M16 |
| 15.3 | Local development setup (LocalStack, Docker Compose) | ✅ | M | Not Started | Platform | 1 | M16 |
| 15.4 | Environment strategy (dev, staging, production) | ✅ | M | Not Started | Platform | 1 | M16 |
| 15.5 | API documentation (OpenAPI/Swagger) | ✅ | C | Not Started | All teams | 3 | C2 |
| 15.6 | Feature flags / experimentation platform | ✅ | C | Not Started | Platform | 3 | C3 |

---

## 16. Best Practices — Reliability

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 16.1 | Multi-AZ deployment | ✅ | M | Not Started | Platform | 3 | M17 |
| 16.2 | Retry with exponential backoff for external calls | ✅ | M | Not Started | All teams | 2 | M22 |
| 16.3 | Dead-letter queues (DLQ) for failed async events | ✅ | S | Not Started | Payments | 3 | S3 |
| 16.4 | Idempotency on all write operations | ✅ | M | Not Started | Deposits / Payments | 2 | M22 |
| 16.5 | Circuit breaker for third-party integrations | ✅ | C | Not Started | All teams | 3 | C8 |
| 16.6 | Automated database backups (point-in-time recovery) | ✅ | M | Not Started | Platform | 3 | M17 |
| 16.7 | Disaster recovery plan (RPO/RTO targets) | ✅ | M | Not Started | Platform | 3 | M17 |
| 16.8 | Health checks and auto-healing | ✅ | M | Not Started | Platform | 3 | M17 |

---

## 17. Best Practices — Performance Efficiency

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 17.1 | Right compute model: Lambda for event-driven, ECS/Fargate for batch | ✅ | M | Not Started | Platform | 1 | M16 |
| 17.2 | Caching (ElastiCache/Redis) for hot data | ✅ | S | Not Started | Platform | 2 | Phase 2 |
| 17.3 | Database optimization (indexing, read replicas, connection pooling) | ✅ | M | Not Started | Deposits / Payments | 2 | M5, M6 |
| 17.4 | Async processing via queues (SQS, EventBridge) | ✅ | M | Not Started | Payments | 2 | M8, M11 |
| 17.5 | Auto-scaling policies | ✅ | C | Not Started | Platform | 3 | C4 |
| 17.6 | API response time SLAs (< 200ms balance, < 2s onboarding) | ✅ | M | Not Started | All teams | 3 | Phase 3 |
| 17.7 | Pagination for large datasets (transaction history) | ✅ | M | Not Started | Deposits | 2 | M12 |

---

## 18. Best Practices — Cost Optimization

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 18.1 | Serverless compute (Lambda, Fargate) | ✅ | M | Not Started | Platform | 1 | M16 |
| 18.2 | Right-size DB instances / Aurora Serverless | ✅ | C | Not Started | Platform | 3 | Phase 3 |
| 18.3 | S3 lifecycle policies for CAMT archival | ✅ | C | Not Started | Payments | 3 | C5 |
| 18.5 | Cost allocation tags | ✅ | C | Not Started | Platform | 3 | C6 |
| 18.6 | AWS Budgets & billing alarms | ✅ | C | Not Started | Platform | 3 | C6 |
| 18.7 | Resource review (Trusted Advisor, Cost Explorer) | ✅ | C | Not Started | Platform | 4 | Phase 4 |

---

## 19. Best Practices — Sustainability

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 19.1 | Minimize over-provisioning (auto-scaling, serverless) | ✅ | C | Not Started | Platform | 3 | C4 |
| 19.2 | Use AWS managed services (RDS, SQS, ElastiCache) | ✅ | M | Not Started | Platform | 1 | M16 |
| 19.3 | Schedule batch jobs during off-peak hours | ✅ | C | Not Started | Payments | 2 | Phase 2 |
| 19.4 | Data retention policies | ✅ | C | Not Started | Platform | 3 | C5 |

---

## 20. Go-Live Changes & Planning

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 20.3 | Data migration / schema changes | ✅ | M | Not Started | All teams | 1 | Phase 1 |
| 20.6 | Risk assessment & mitigation strategies | ✅ | M | Not Started | Architect | 1 | Roadmap §5 |
| 20.7 | Team capacity & skill requirements | ✅ | M | Not Started | Architect | 1 | Roadmap §1 |

---

## 21. Nice to Have Deliverables

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 21.1 | Database entity-relationship diagram | ✅ | C | Not Started | Architect | 2 | C1 |
| 21.2 | ER: core entities, relationships, key fields | ✅ | C | Not Started | Architect | 2 | C1 |
| 21.3 | ER: audit trail strategy (separate tables or event sourcing) | ✅ | C | Not Started | Architect | 2 | C1 |
| 21.4 | Sample API documentation (OpenAPI/Swagger) | ✅ | C | Not Started | All teams | 3 | C2 |
| 21.5 | API: request/response schemas with sample payloads | ✅ | C | Not Started | All teams | 3 | C2 |
| 21.6 | API: auth requirements, error codes, pagination | ✅ | C | Not Started | All teams | 3 | C2 |
| 21.7 | API: versioning strategy | ✅ | C | Not Started | All teams | 3 | C2 |
| 21.8 | API: REST vs GraphQL consideration | ✅ | C | Not Started | Architect | 1 | C2 |
| 21.9 | Stakeholder questions (volume, multi-currency, regulatory) | ✅ | C | Not Started | Architect | 1 | Phase 1 |

## 22. Regulatory Reporting

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 22.1 | Transaction reports to central bank / financial authority | ❌ | M | Not Started | Payments | 2 | M30 |
| 22.2 | AML suspicious transaction monitoring | ✅ | M | Not Started | Onboarding | 2 | M31 |

---

## 23. Customer Experience & Notifications

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 23.1 | Email/SMS for deposit received, maturity, payout, rate change, compliance | ✅ | M | Not Started | Deposits | 2 | M33 |
| 23.2 | Legal documents: T&C, privacy policy, fee schedule presented | ❌ | M | Not Started | Onboarding | 1 | M38 |

---

## 24. Operations & Back-Office Tooling

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 24.1 | Admin tool for setting/changing interest rates, effective date logic | ✅ | M | Not Started | Deposits | 2 | M35 |
| 24.2 | Account closure flow: customer-initiated full account closure | ❌ | M | Not Started | Deposits | 2 | M37 |
| 24.3 | Back-office admin portal: view accounts, resolve issues, trigger payouts | ✅ | S | Not Started | Deposits | 3 | S4 |
| 24.4 | Customer support tooling: view customer journey, identify drop-offs | ✅ | S | Not Started | Onboarding | 3 | S5 |

---

## 25. Core Financial & Architectural Strategy

| # | Requirement | Covered | MoSCoW | Status | Team | Phase | Roadmap Ref |
|---|-------------|---------|--------|--------|------|-------|-------------|
| 25.1 | Data residency & GDPR: data stays in AWS region, classification policy | ✅ | M | Not Started | Platform | 1 | M34 |
| 25.3 | Vendor fallback strategy: alternatives for IDV/PEP/bank validation | ✅ | S | Not Started | Onboarding | 3 | S8 |
| 25.4 | Event sourcing / CQRS for financial transaction history | ✅ | C | Not Started | Deposits | 1 | C10 |
| 25.5 | BFF (Backend for Frontend) pattern for web app APIs | ✅ | C | Not Started | Architect | 1 | C11 |
| 25.6 | Contract testing between services | ❌ | C | Not Started | Platform | 3 | C12 |
| 25.7 | API versioning strategy | ❌ | C | Not Started | Architect | 1 | C13 |

