# Alborz Bank — Savings Account: Team Structure

**Total headcount: ~16 people** (14 in teams + 1 Solution Architect + 1 Product Lead)

---

## Team 1 — Onboarding & Compliance (4 people)

Owns the customer-facing onboarding flow and all compliance integrations. All engineers share QA and testing responsibilities.

| Role | Count | Key Skills |
|------|-------|------------|
| Backend Engineer | 2 | Node.js/TypeScript, AWS Lambda, DynamoDB/RDS, API testing, compliance scenario testing |
| Frontend Engineer | 1 | React/Next.js, TypeScript, responsive design, E2E testing |
| Product Manager / Owner | 1 | Banking/fintech domain knowledge, regulatory awareness, sprint delivery & coordination |

**Tech Stack:**
- **Languages:** Node.js, TypeScript, React
- **Compute & Data:** AWS Lambda, Amazon DynamoDB
- **Integrations:** Onfido/IDnow (IDV), Plaid/Tink (Bank Validation), ComplyAdvantage (PEP/Sanctions), Zendesk (CRM)
- **Testing:** Jest, Cypress

**Scope:**
- Customer registration and identity verification (IDnow/Onfido integration)
- Bank account validation (IBAN check, ownership verification via Plaid/Tink)
- PEP/Sanctions screening (ComplyAdvantage/World-Check integration)
- KYC questionnaire engine (configurable questions, admin UI)
- Customer state machine (DRAFT → VERIFIED → ACTIVE / REJECTED)
- Customer eligibility validation (age, residency, entity type)
- AML suspicious transaction monitoring
- Legal documents presentation (T&C, privacy policy)
- Customer support tooling
- Vendor fallback strategy

**Roadmap items owned:** M1, M2, M3, M4, M25, M28, M31, M38, M39, S2, S5, S8, C11 (shared)

---

## Team 2 — Deposits & Transactions (4 people)

Owns the core banking logic: deposit accounts, transactions, and the customer web application. All engineers share QA and testing responsibilities.

| Role | Count | Key Skills |
|------|-------|------------|
| Backend Engineer | 2 | Node.js/TypeScript, AWS Lambda, SQS, financial calc testing, edge case validation |
| Frontend Engineer | 1 | React/Next.js, TypeScript, data-heavy UI, E2E testing, UI automation |
| Product Manager / Owner | 1 | Banking product knowledge, deposit lifecycle understanding, sprint delivery & coordination |

**Tech Stack:**
- **Languages:** Node.js, TypeScript, React, Next.js
- **Compute & Data:** AWS Lambda, Amazon Aurora (PostgreSQL)
- **Communications:** AWS SES (Email), AWS SNS (SMS)

**Scope:**
- Fixed term deposit and overnight savings account creation and management
- Transaction processing (PYI, PYO, CAN, IBR, TAX)
- Customer web app: account dashboard, transaction history, withdrawal requests
- Interest rate configuration and management
- Idempotency on deposit write operations
- Customer notifications (Email/SMS)
- Interest rate management admin tool
- Account closure
- Back-office admin portal
- **Customer Web BFF (C11)**: Aggregating account, transaction, and onboarding data for the frontend.

**Roadmap items owned:** M5, M6, M7, M12, M22 (shared), M33, M35, M37, S1, S4, C10, C11 (owner)

---

## Team 3 — Payments & Accounting (3 people)

Owns the backend processing: CAMT file ingestion, interest/tax calculation, and payout execution. All engineers share QA and testing responsibilities.

| Role | Count | Key Skills |
|------|-------|------------|
| Backend Engineer | 2 | Node.js/TypeScript, SFTP, batch processing, SQS, batch job testing, data integrity validation |
| Data/Financial Engineer | 1 | Financial calculations, accounting principles, reconciliation checks |

**Tech Stack:**
- **Languages:** Node.js, TypeScript, XML Parsing (CAMT.053)
- **Orchestration:** AWS Lambda, AWS Step Functions, EventBridge (Cron)
- **Storage/Data/Integration:** Amazon S3, Amazon DynamoDB (Locks), AWS SQS, AWS Transfer Family (SFTP)

**Scope:**
- CAMT file parser (SFTP polling, ISO 20022 XML parsing, reference matching)
- Interest calculation engine (daily for overnight, fixed for term deposits)
- Tax calculation engine (withholding tax computation)
- Payout processing job (PYO/CAN execution, bank transfer initiation)
- Reconciliation and reporting
- Idempotency on payment write operations
- Regulatory reporting (transaction reports)

**Roadmap items owned:** M8, M9, M10, M11, M22 (shared), M30, S3

---

## Team 4 — Platform & Infrastructure (3 people)

Owns shared infrastructure, CI/CD, security, monitoring, and cross-cutting concerns.

| Role | Count | Key Skills |
|------|-------|------------|
| Platform/DevOps Engineer | 2 | AWS CDK, CloudFormation, CI/CD (GitHub Actions/CodePipeline), Terraform |
| Security Engineer | 1 | AWS security services (KMS, WAF), Auth-Service, PII handling, compliance |

**Tech Stack:**
- **Infrastructure as Code:** AWS CDK, CloudFormation
- **Security & Data:** Centralized Identity Broker / API Gateway, Amazon DynamoDB, AWS KMS, AWS WAF, AWS Secrets Manager, SSM Parameter Store
- **Observability:** Amazon CloudWatch, AWS Systems Manager, Alarms, ELK Stack
- **Resilience/Testing:** AWS SQS (DLQ), K6/Artillery (Load Testing), OWASP Tools

**Scope:**
- AWS infrastructure provisioning (CDK stacks for all teams)
- CI/CD pipelines with automated quality gates
- Authentication & authorization (Brokering IdPs & Central API Gateway)
- Encryption (KMS), secrets management (Secrets Manager, SSM)
- Monitoring & alerting (CloudWatch, Grafana, CloudTrail)
- Logging infrastructure (ELK stack, structured logging, PII redaction, audit trail)
- Environment management (dev, staging, production)
- Rate limiting, API security hardening
- Data residency & GDPR compliance
- Contract testing
- Back-office Operations BFF

**Roadmap items owned:** M13, M14, M15, M16, M17, M18, M19, M20, M21, M23, M24, M26, M34, C12, C11 (shared)

---

## Cross-Team Roles

| Role | Count | Responsibility |
|------|-------|----------------|
| Solution Architect | 1 | Cross-team technical design, integration strategy, architecture diagrams (M29), event sourcing / CQRS (C10), BFF pattern (C11), API versioning (C13) |
| Product Lead | 1 | Overall product vision, cross-team roadmap prioritization, stakeholder management |
---

## Summary

| Team | People | Focus Area |
|------|--------|------------|
| Onboarding & Compliance | 4 | Registration, KYC, PEP/Sanctions, legal docs, reporting |
| Deposits & Transactions | 4 | Core banking, web app, transactions, interest rates, ops |
| Payments & Accounting | 3 | CAMT, interest/tax calculation, payouts, regulatory reporting |
| Platform & Infrastructure | 3 | AWS, CI/CD, security, monitoring, GDPR |
| Cross-team roles | 2 | Architecture, strategic design, overall product vision |
| **Grand Total** | **16** | |
