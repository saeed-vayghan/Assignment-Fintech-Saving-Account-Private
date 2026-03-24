# Alborz Bank — Savings Account Product Roadmap

**Timeline:** 12 months to go live

**Tech Stack:** Node.js / TypeScript, AWS (serverless-first), CDK / CloudFormation

**Methodology:** Agile (2-week sprints), cross-functional teams

---

## 1. Team Structure

Team definitions, roles, skills, and scope are documented in teams.md.

**Quick summary:** 4 teams (16 people total)
- Onboarding & Compliance (4)
- Deposits & Transactions (4)
- Payments & Accounting (3)
- Platform & Infrastructure (3)
- plus 1 Solution Architect
- plus 1 Product Lead

---

## 2. MoSCoW Prioritization

### Must Have (MVP — Without these, the product cannot launch)

| # | Feature | Owner Team |
|---|---------|------------|
| M1 | Customer registration & identity verification | Onboarding |
| M2 | Bank account validation (IBAN + ownership) | Onboarding |
| M3 | PEP/Sanctions screening with compliance review | Onboarding |
| M4 | KYC questionnaire (5 questions, configurable) | Onboarding |
| M5 | Fixed term deposit: create, mature, rollover | Deposits |
| M6 | Overnight savings account: create, top-up, partial withdraw | Deposits |
| M7 | Transaction processing: PYI, PYO, CAN, IBR, TAX | Deposits |
| M8 | CAMT file import from SFTP (parse, match, create PYI) | Payments |
| M9 | Interest calculation engine (fixed + variable rates) | Payments |
| M10 | Tax calculation engine (withholding tax) | Payments |
| M11 | Payout processing job (PYO for matured/withdrawn, CAN for cancellations) | Payments |
| M12 | Customer web app: account dashboard & transaction history | Deposits |
| M13 | Authentication & authorization (Auth-Service) | Platform |
| M14 | Encryption at rest (KMS) and in transit (TLS) | Platform |
| M15 | Audit logging for all compliance actions | Platform |
| M16 | CI/CD pipeline & infrastructure as code (CDK) | Platform |
| M17 | Production environment setup (multi-AZ, backups) | Platform |
| M18 | Monitoring dashboards (CloudWatch + Grafana) | Platform |
| M19 | ELK stack logging (centralized log aggregation & search) | Platform |
| M20 | Alerting strategy (basic thresholds and alarms) | Platform |
| M21 | Rate limiting & API security hardening | Platform |
| M22 | Idempotency on all write operations | Deposits / Payments |
| M23 | Structured logging with correlation IDs | Platform |
| M24 | PII redaction in logs | Platform |
| M25 | Customer eligibility validation (age ≥ 18, residency, entity type) | Onboarding |
| M26 | Secrets management (AWS Secrets Manager, SSM Parameter Store) | Platform |
| M28 | Customer state machine (DRAFT → PENDING_VERIFICATION → ACTIVE / REJECTED) | Onboarding |
| M29 | Architecture diagrams: onboarding sequence + customer web app process flow | Architect |
| M30 | Regulatory reporting: transaction reports to central bank / financial authority | Payments |
| M31 | Regulatory reporting: AML suspicious transaction monitoring | Onboarding |
| M33 | Customer notifications: email/SMS for deposit received, maturity, payout, rate change, compliance status | Deposits |
| M34 | Data residency & GDPR compliance: customer data stays within designated AWS region, data classification policy | Platform |
| M35 | Interest rate management: admin tool for setting/changing rates, effective date logic, rate change history | Deposits |
| M37 | Account closure flow: customer-initiated full account closure | Deposits |
| M38 | Legal documents: Terms & Conditions, privacy policy, fee schedule presented during onboarding | Onboarding |
| M39 | KYC Webhook FIFO Queue: Immediate ingestion of IDV webhooks to SQS FIFO to prevent data loss | Onboarding |

### Should Have (Important but launch can proceed without if delayed)

| # | Feature | Owner Team |
|---|---------|------------|
| S1 | Customer web app: withdrawal request flow | Deposits |
| S2 | Compliance officer dashboard (review PEP flags, approve/reject) | Onboarding |
| S3 | DLQ handling & automated retry for failed payments | Payments |
| S4 | Back-office admin portal: view customer accounts, resolve issues, trigger manual payouts | Deposits |
| S5 | Customer support tooling: view customer journey, identify drop-off points | Onboarding |
| S8 | Vendor fallback strategy: alternative providers if primary IDV/PEP/bank validation provider is down | Onboarding |

### Could Have (Nice additions if time and budget allow)

| # | Feature | Owner Team |
|---|---------|------------|
| C1 | Database ER diagram & formal data model documentation | Architect |
| C2 | Sample API documentation (OpenAPI/Swagger) | All teams |
| C3 | Feature flags / experimentation platform | Platform |
| C4 | Auto-scaling policies (queue-depth, CPU-based) | Platform |
| C5 | S3 lifecycle policies for CAMT archival (IA → Glacier) | Payments |
| C6 | Cost allocation tags & AWS Budget alarms | Platform |
| C7 | Runbooks for common operational tasks | Platform |
| C8 | Circuit breaker pattern for third-party integrations | All teams |
| C10 | Event sourcing / CQRS for financial transaction history | Architect |
| C11 | BFF (Backend for Frontend) pattern: separate API for web app vs. internal services | Deposits / Platform |
| C12 | Contract testing between services | Platform |
| C13 | API versioning strategy (URL path or header-based) | Architect |

### Won't Have (Out of scope for this 12-month delivery)

| # | Feature | Reason |
|---|---------|--------|
| W1 | Multi-currency support | Single market launch only |
| W2 | Mobile app (iOS/Android) | Web-first approach; mobile can follow post-launch |
| W3 | Business/trust/minor accounts | Only individual adult residents in scope |
| W4 | Real-time payment rails (instant SEPA) | CAMT batch processing is sufficient for V1 |
| W5 | Advanced analytics / BI dashboards | Post-launch improvement |
| W6 | Multi-language / i18n support | Single market = single language for V1 |
| W7 | Joint accounts | V2 product expansion |
| W8 | Higher deposit tiers with differentiated rates | V2 product expansion |
| W9 | Integration with other bank products (loans, credit cards) | V2 cross-selling |
| W10 | Multi-region / international expansion | V2 geographic expansion |
| W11 | Real-time push notifications (WebSocket/SSE) | Email/SMS sufficient for V1 |

---

## 3. Phased Roadmap (12 Months)

### Phase 1 — Foundation (Months 1–3)

**Goal:** Set up infrastructure, core data models, and the onboarding flow.

| Team | Month 1 | Month 2 | Month 3 | Tech Stack |
|------|---------|---------|---------|------------|
| **Platform** | AWS infra (CDK, Auth, KMS) | CI/CD pipelines, staging envs, logging | Dashboards, ELK, alerts, PII, audit logs | AWS CDK, KMS, Auth-Service, DynamoDB, EventBridge, ELK |
| **Onboarding** | Data model, APIs, registration | ID verification (Onfido/IDnow) | Bank validation, PEP screening | Node.js, Lambda, DynamoDB, Onfido & Plaid APIs |
| **Deposits** | DB schema, core data model | Fixed term deposit logic | Overnight savings logic | Node.js, Lambda, Aurora (PostgreSQL) |
| **Payments** | Mock CAMT generator | ISO 20022 parsing rules | SFTP setup with partner bank | Node.js, XML Parsing, AWS Transfer Family (SFTP) |

> 🏛️ **Architect Focus:** Guiding all teams strictly to the defined models and standards. Focusing on initial cross-team data model definitions, identity integration security patterns, and EventBridge messaging schemas.

**Milestone:** Onboarding flow works end-to-end in staging. Core deposit accounts can be created via API. Monitoring dashboards, ELK stack, alerting, and structured logging are operational.

---

### Phase 2 — Core Processing (Months 4–6)

**Goal:** Build the payments pipeline, interest/tax engines, and the customer web app.

| Team | Month 4 | Month 5 | Month 6 | Tech Stack |
|------|---------|---------|---------|------------|
| **Payments** | SFTP, CAMT parser, PYI pipeline | Interest engines, IBR txns | Tax engine, payout jobs | Node.js, Lambda, S3, DynamoDB, SQS, EventBridge |
| **Deposits** | Core transaction processing | Customer web app (dashboard) | Customer web app (txns) | Node.js, Lambda, Aurora (PostgreSQL), React, Next.js |
| **Onboarding** | KYC questionnaire engine | Onboarding E2E testing | Compliance UI dashboard | React, Node.js, Jest/Cypress |
| **Platform** | Rate limiting, WAF, CORS | API security & pen test prep | Validate MVP completeness | AWS WAF, API Gateway, OWASP Security Tools |

> 🏛️ **Architect Focus:** Ensuring data flow and API contracts align across teams. Conducting pre-audit security architecture reviews, defining the API versioning strategy, and validating idempotency implementations.

**Milestone:** Full deposit lifecycle works: onboard → deposit → accrue interest → calculate tax → payout. Customer web app shows accounts and transactions. Rate limiting and API security hardening in place. Idempotency enforced on all write operations.

---

### Phase 3 — Integration & Hardening (Months 7–9)

**Goal:** End-to-end integration testing, security hardening, reliability improvements.

| Team | Month 7 | Month 8 | Month 9 | Tech Stack |
|------|---------|---------|---------|------------|
| **All teams** | E2E integration & edge case testing | Load testing & pen testing | Bug fixing, final hardening | Jest, Cypress, Postman, Load Testing Tools |
| **Platform** | Setup load testing infra | DLQ & retry mechanisms | Setup Production (multi-AZ) | AWS SQS DLQ, Multi-AZ RDS/Aurora, K6/Artillery |
| **Payments** | Reconciliation checks | Test payment edge cases | Data integrity validation | Custom SQL, Node.js scripts |
| **Deposits** | Validate interest math | T&C document templates | Verify notification payloads | AWS SES, SNS, PDF Generation Libraries |
| **Onboarding**| Vendor fallback testing | Tune IDV webhooks | Refine support tooling | React, Circuit Breaker Libraries |

> 🏛️ **Architect Focus:** Driving integration consistency and resolving edge cases. Assisting teams in addressing structural findings from penetration tests, rigorously reviewing failure scenarios and fallback logic, and finalizing production architecture diagrams.

**Milestone:** System passes security audit and performance benchmarks. All services deployed to production-like environment. DLQ and retry mechanisms tested.

---

### Phase 4 — Launch Preparation (Months 10–12)

**Goal:** UAT, operational readiness, soft launch, and GA release.

| Team | Month 10 | Month 11 | Month 12 | Tech Stack |
|------|----------|----------|----------|------------|
| **All teams** | UAT, bug fixes, final docs | Soft launch (invite-only) | 🚀 GA Launch (Full rollout) | Production Metrics, Datadog/CloudWatch |
| **Platform** | Operational readiness review | Runbooks, tuning | Post-launch support | AWS Trusted Advisor, Systems Manager, Alarms |
| **Payments** | Partner UAT coordination | Validate Prod CAMT | First EOM batch support | AWS SFTP, XML Parsers, Step Functions |
| **Deposits** | Account closure flow | Live transaction triage | Post-launch support | CloudWatch Dashboards, Admin UI |
| **Onboarding**| Train support agents | Monitor soft-launch funnel | Scale IDV vendor tier | CRM, Google Analytics/Mixpanel |

> 🏛️ **Architect Focus:** Overseeing operational readiness. Reviewing UAT feedback for necessary architectural tweaks, providing the final technical Go/No-Go sign-off, and leading the post-launch retrospective & V2 planning.

**Milestone:** Product is live, processing real deposits and payments, with monitoring and on-call support active.

---

## 4. Key Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Third-party provider delays (Onfido, ComplyAdvantage) | Blocks onboarding flow | Start integration early (Month 2), have mock services for dev |
| CAMT file format variations | Breaks payment import | Get sample files from the bank early, build extensive parser tests |
| Regulatory requirements change | Scope creep in compliance | Keep KYC questionnaire configurable, involve compliance team in each phase review |
| Underestimated interest/tax calculation complexity | Delays in Phase 2 | Involve a financial domain expert, validate formulas with business early |
| Team ramp-up time | Slow start in Phase 1 | Front-load hiring, pair new members with experienced engineers |
| GDPR / data residency non-compliance | Legal action, launch blocked | Engage legal team in Phase 1, confirm AWS region, classify all data early |
| Missing regulatory reports at launch | Regulator blocks go-live | Identify all required reports early (Phase 1), build reporting pipeline in Phase 2 |
| Vendor lock-in or outage (IDV/PEP providers) | Service downtime | Architect provider-agnostic interfaces, evaluate fallback providers |
| Interest rounding discrepancies | Financial audit failures | Define rounding rules and day count conventions in Phase 1, validate with business |
| Key team member leaves mid-project | Knowledge loss, delays | Document decisions in ADRs, ensure no single point of failure per team |

---

## 5. Phase Gate Criteria

Before moving to the next phase, the following must be met:

| Gate | Criteria |
|------|----------|
| **Phase 1 → 2** | Onboarding E2E works in staging. Deposit accounts created via API. CI/CD operational. Monitoring, ELK, alerting, structured logging, PII redaction all operational. Data residency and GDPR compliance confirmed. |
| **Phase 2 → 3** | Full deposit lifecycle works E2E. Web app shows accounts and transactions. CAMT pipeline processes test files. Rate limiting and API security hardened. Idempotency enforced. Interest rate admin tool operational. Notification service sending emails/SMS. Regulatory report templates defined. |
| **Phase 3 → 4** | Security audit passed. Performance benchmarks met. DLQ/retry mechanisms tested. Production environment provisioned. Regulatory reports generate correctly with test data. Customer legal documents (T&C, privacy policy) approved. |
| **Phase 4 → Launch** | UAT signed off by business. ORR checklist completed. On-call rotation active. Runbooks documented. Regulatory reporting pipeline validated with real data. Customer notification flows verified end-to-end. |
