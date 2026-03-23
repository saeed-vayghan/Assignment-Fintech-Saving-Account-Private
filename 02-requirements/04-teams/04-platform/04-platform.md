# Team 4: Platform & Infrastructure

### **Purpose**:
Provide the foundational cloud infrastructure (M16, M17), shared security services (M13, M14, M26), CI/CD pipelines (M16), and centralized observability (M18, M19) for all feature teams. Ensures strict adherence to Data Residency and GDPR rules (M34).

### **UI Applications**:
No custom UI built by this team. They manage third-party infrastructure operation UIs like AWS Console, Grafana dashboards (M18), and Kibana (M19).

### **User-Facing Service**:
No. Acts primarily as a technical backbone for the internal feature teams.

### **Authentication / Authorization Mechanisms**:
*(See `06-auth-service.md` for Centralized Issuance / Decentralized AuthZ strategy)*
- **Central Identity Broker:** Configures the actual AuthN providers (`eID`, `Local-Cred`, `SSO`) but relies on external IdPs for federated identity persistence.
- **Central API Gateway AuthZ:** Acts as the zero-trust front door, enforcing cryptographic validation of `OAuth-Token` and `OAuth-RBAC` before injecting the `customerId` header and securely routing to domain teams.
- **CI/CD & Queues:** Secured internally via `AWS-IAM` (OIDC).

### **Microservices & Infrastructure**:
  1. `Auth-Service`: Issues and validates authentication tokens for the entire ecosystem (M13).
  2. `Central API Gateway`: The unified entry portal tracing and routing traffic to downstream teams, enforcing rate limiting and strict CORS/API security hardening (M21) alongside encryption in transit (M14).
  3. `Central Observability Pipeline`: Manages the Amazon Kinesis Data Firehose streams that buffer and ship millions of structured CloudWatch logs into the ELK stack in near real-time (M19).
  4. `Data Residency Controls`: Ensures all architectural footprints strictly enforce that customer data stays within the designated AWS region (M34).
  5. `Back-office Operations BFF`: A standalone serverless API layer aggregating compliance flags, support CRM data, and audit logs for internal employees (C11).
  6. `Notification Engine`: Centralized async listener that integrates with AWS SES/SNS to fire localized SMS and Emails (M33) in response to cross-domain Pub/Sub events.

### **Databases & Storage Access**:

| Database / Storage | Type | Access Level | Purpose / Justification |
|--------------------|------|--------------|-------------------------|
| **Amazon DynamoDB** (Auth Service) | NoSQL Key-Value | **Read / Write** (Owned) | **Ultra-Low Latency:** Single-digit millisecond response times for fetching user credentials (M13) without connection pooling bottlenecks during massive traffic spikes. |
| **ELK Stack** (Audit / Operational Logs) | Search Index | **Read / Write** (Owned) | **Full-Text Search:** Automatically indexes every field, enabling fast search across millions of compliance events using `correlationId` or `customerId` (M15, M19). |

### **Caching Layer**:
No traditional data cache, but acts as the CDN / Edge caching layer via AWS WAF and API Gateway.

### **Message Queue / Async Comm**:
Manages the provisioning of the central `AWS EventBridge` bus that all teams use to communicate with each other asynchronously.

- Uses **Amazon Kinesis Data Firehose** to asynchronously buffer and stream compliance and operational logs into OpenSearch without dropping payloads during massive traffic spikes. Or any other AWS based simpler solution.

### **External Systems**:
AWS SES (Simple Email Service) and SNS (Simple Notification Service) for global transactional message delivery.

### **Third-Party Services**:
None. Relies purely on managed AWS infrastructure services (KMS for M14 encryption at rest, AWS Secrets Manager & SSM for M26).

### **Other Team Interactions**:
  - **All Teams**: Every team is required to use Platform's Auth-Service (M13), API Gateway (M21), AWS CDK deployment constructs (M16), and ELK Logging pipeline (M19). The Platform team also enforces contract testing between all these services (C12).

### **Edge Cases & Failure Scenarios**:

| Category | Scenario | Immediate Strategy / Impact | HITL Action / Threshold |
|----------|----------|-----------------------------|-------------------------|
| **Retryable** | **API Surge / DDoS Attempt:** Massive traffic hits public endpoints. | AWS WAF rate-limiting intercepts and temporarily blocks IPs. | **SecOps (Major):** If surge sustains >15 mins, investigate if manual IP blacklisting is needed. |
| **Retryable** | **Auth-Service Latency:** DynamoDB throttles reads on credential fetching. | Service degrades, client SDKs auto-retry authentication. | **Platform Ops (Moderate):** CloudWatch alarms trigger to manually adjust DB capacity if needed. |
| **Moderate** | **Certificate Expiry / Secret Rotation Warning:** Automated KMS/TLS expiry nears. | Automation generates high-priority Slack/PagerDuty alerts. | **Platform Engineer:** Verifies seamless rotation deploys correctly to prevent strict TLS outages. |
| **Moderate** | **ELK Cluster Storage 90% Full:** Logging volume unexpectedly spikes. | ElasticSearch shifts older indices to sluggish cold storage. | **Platform Ops:** Provisions more EBS volumes or tunes the retention policy. |
| **Major** | **AWS Region Degradation:** Entire eu-central-1 AWS zone experiences packet loss. | Widespread degradation across all teams and services. | **Platform Leadership:** Makes executive decision to initiate Multi-AZ/DR runbooks (M17). |
| **Major** | **Compromised Token:** Suspicious activity from a token. | API Gateway flags irregular access patterns. | **SecOps:** Immediately revokes Tokens, isolates internal network, and triggers incident response. |

### **Logs and Metrics**:
Owns the centralized ingestion pipeline for all structured logging (M23), CloudWatch metrics/dashboards (M18), strict audit logging for compliance actions (M15), alerting strategies (M20), and enforces global PII redaction across all logs automatically (M24).

### **API Exposed**:
  - Public-facing Auth-Service REST APIs (for user authentication)
  - Reverse-proxy routing APIs (API Gateway) acting as the front door for all other teams.

### **Roadmap & MoSCoW Prioritization References**:
#### **Must Have (MVP)**
- `M13`: Authentication & authorization (Auth-Service)
- `M14`: Encryption at rest (KMS) and in transit (TLS)
- `M15`: Audit logging for all compliance actions
- `M16`: CI/CD pipeline & infrastructure as code (CDK)
- `M17`: Production environment setup (multi-AZ, backups)
- `M18`: Monitoring dashboards (CloudWatch + Grafana)
- `M19`: ELK stack logging (centralized log aggregation & search)
- `M20`: Alerting strategy (basic thresholds and alarms)
- `M21`: Rate limiting & API security hardening
- `M23`: Structured logging with correlation IDs
- `M24`: PII redaction in logs
- `M26`: Secrets management (AWS Secrets Manager, SSM Parameter Store)
- `M34`: Data residency & GDPR compliance

#### **Could Have**
- `C12`: Contract testing between services
