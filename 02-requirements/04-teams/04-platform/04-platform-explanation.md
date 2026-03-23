# Platform Team: Architecture Explanation & Nuances

This document explains the architectural decisions behind dividing the Platform team's domain into the specific infrastructure footprints exactly as defined in the `04-platform.md` profile.

### The 5 Core Pillars:
1. `Auth-Service (Central Identity Broker)`
2. `Central API Gateway`
3. `Central Observability Pipeline (ELK & Firehose)`
4. `Data Residency Controls & Cross-Cutting Infrastructure`
5. `Back-office Operations BFF`
6. `Centralized Notification Engine (SES/SNS)`

---

## 1. Are they Serverless?
**Heavily Native.** The Platform team primarily writes **Infrastructure as Code (AWS CDK)** to provision managed AWS services rather than writing massive Node.js applications. 

The API Gateway is a native AWS service, Auth-Service utilizes Cognito or lightweight edge Lambdas against DynamoDB, and the observability stack relies on managed OpenSearch (ELK). Their goal is to write as little custom server compute as possible, relying entirely on the cloud provider's managed SLAs.

---

## 2. Why exactly these 6 boundaries? (The Challenge)

Let's challenge if we have too many or too few services by evaluating their Domain-Driven Design (DDD) boundaries and failure domains.

### 1. Auth-Service (Central Identity Broker)
* **Why it exists:** To act as the single source of truth for all Authentication (`eID`, `Local-Cred`, `SSO`).
* **Challenge:** *Why not let Onboarding handle Customer Login, and a separate internal IT team handle Employee Login?* **Security Auditing.** Building cryptographic verification, token issuance, and password hashing is high-risk. If two different teams build it, the bank doubles its attack surface area. By centralizing Identity Brokering into Platform, the bank can afford to have dedicated Security Engineers rigorously pen-test a single entry point.

### 2. Central API Gateway
* **Why it exists:** The unified entry portal tracing and routing traffic to downstream teams, enforcing rate limiting (WAF), and conducting the first layer of zero-trust `OAuth-Token` validation.
* **Challenge:** *Why not let each microservice have its own public ALB or API Gateway?* **Governance and Protection.** If the Deposits ledger API was accidentally exposed to the public internet because a junior developer misconfigured their CDK stack, it would be catastrophic. By forcing all external traffic exclusively through a Platform-owned centralized Gateway, the Platform team guarantees that rate-limiting, DDoS protection, and token validation simply cannot be bypassed.

### 3. Central Observability Pipeline (ELK & Firehose)
* **Why it exists:** To aggregate millions of compliance, security, and operational logs from every microservice across the bank into a single, searchable OpenSearch cluster. It uses Kinesis Data Firehose to synchronously buffer the logs.
* **Challenge:** *Why not just let each Lambda write directly to Elasticsearch?* **Connection pooling and decoupling.** If 10,000 Lambdas spin up and try to establish direct HTTP connections to write logs to ElasticSearch, the cluster will crash. By pushing logs asynchronously to CloudWatch, and using Firehose to batch and stream them to ELK, the logging pipeline is completely decoupled from the synchronous compute layer.

### 4. Data Residency & Cross-Cutting Infrastructure
* **Why it exists:** To provide CI/CD, KMS Encryption, SQS/EventBridge, and strict data-locality rules.
* **Challenge:** *Why not let teams provision their own databases and queues however they want?* **Compliance.** A heavily regulated bank requires that all PII is encrypted at rest using specific KMS keys, logs are stripped of PII before indexing, and data never leaves `eu-central-1` (GDPR). If domain teams had raw AWS root access, achieving this compliance would require manual code reviews. By forcing teams to use Platform's IaC modules, compliance is achieved automatically through technical mandates.

### 5. Back-office Operations BFF
* **Why it exists:** To aggregate sensitive administrative data (Compliance flags, AML hits, Audit logs) for internal employee use while enforcing strict **RBAC (C11)**.
* **Challenge:** *Why not let the individual team BFFs handle their own admin views?* **Security Consolidation.** Internal bank employee tools often need data from *every* team (e.g., seeing a customer's Onboarding status AND their Deposit history). By centralizing this into a single Platform-managed BFF, we can ensure that PII redaction and audit logging are applied consistently across all internal dashboards, regardless of which team's data is being viewed.

### 6. Centralized Notification Engine
* **Why it exists:** To act as the single outbound funnel for all customer SMS and Email communication (M33).
* **Challenge:** *Why not let Onboarding send KYC approval emails, and Deposits send maturity emails?* **Brand Consistency & Spam Prevention.** If every microservice integrates with Twilio independently, the bank loses central control over localized email templates and rate limits (e.g., spamming a user 3 times in one minute). By centralizing notifications, domains simply fire an EventBridge event (`AccountMatured`), and the Notification Engine handles formatting the payload into a beautifully branded HTML email while tracking strict delivery telemetry.

---

## Conclusion
The Platform team operates the **Zero-Trust Boundary and the Regulatory Backbone**. By monopolizing the API Gateway and the Auth-Service, they ensure the blast radius of a malicious attacker never reaches the actual banking domain logic. By owning the IaC pipelines and the cross-domain Back-office BFF, they guarantee systemic compliance and secure internal operations without compromising the velocity of the feature squads.
