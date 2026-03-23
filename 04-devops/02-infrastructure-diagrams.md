# DevOps & AWS Infrastructure Architecture

This document maps out how the DevOps tools and AWS managed services support the Alborz Bank ecosystem, strictly aligned with the **AWS Well-Architected Framework**.

---

## 🏗️ 1. Network & Traffic Flow (Core Infrastructure)

```mermaid
flowchart TD
    classDef secure fill:#f9dbbd,stroke:#d97736,stroke-width:2px,color:#000
    classDef compute fill:#ade8f4,stroke:#0077b6,stroke-width:2px,color:#000
    classDef network fill:#c0efd1,stroke:#264653,stroke-width:2px,color:#000
    classDef storage fill:#e9c46a,stroke:#e76f51,stroke-width:2px,color:#000
    
    Internet((Internet / Customers))
    
    subgraph Edge ["AWS Edge (Security & Routing)"]
        WAF(AWS WAF - Rate Limiting & Rules):::secure
        CloudFront(CloudFront - CDN Caching):::network
        WAF --> CloudFront
    end
    
    Internet --> WAF
    
    subgraph Public Subnet ["Public Subnet"]
        ALB(Application Load Balancer):::network
        APIGW(API Gateway - Mutual TLS):::network
        Auth(Custom Authorizer - IAM / KMS):::secure
        
        CloudFront --> ALB
        ALB --> APIGW
        APIGW --> Auth
    end
    
    subgraph Private VPC ["Private Subnet (Isolated Compute)"]
        direction TB
        ECS[ECS / Fargate Auto Scaling<br/>Synchronous APIs]:::compute
        SQS[Amazon SQS<br/>Async FIFO Buffer DLQs]:::network
        LMB[Lambda Auto Scaling<br/>Event Processors]:::compute
        
        APIGW -->|REST/GraphQL| ECS
        APIGW -->|Webhook Dumps| SQS
        SQS -->|Throttled Polling| LMB
    end
    
    subgraph Data VPC ["Data Subnet (Strict Perimeter)"]
        direction TB
        DB[(Aurora PostgreSQL Multi-AZ / DynamoDB<br/>Encryption via KMS)]:::storage
        S3[(Amazon S3 Intelligent-Tiering<br/>Object Lock Enabled)]:::storage
        
        ECS --> DB
        LMB --> DB
        LMB --> S3
    end
```

### Well-Architected Pillars Covered Here
* **Security:** Strict network segmentation (VPCs), DDoS protection (WAF), and at-rest encryption (KMS).
* **Reliability:** High availability via Multi-AZ deployments and compute protection using SQS FIFO buffers.
* **Performance Efficiency:** Serverless auto-scaling (Fargate/Lambda) and global asset caching (CloudFront).

---

## ⚙️ 2. CI/CD Pipeline Flow (Deployment & Quality)

```mermaid
flowchart LR
    classDef git fill:#2b2d42,stroke:#000,stroke-width:2px,color:#fff
    classDef ci fill:#8d99ae,stroke:#2b2d42,stroke-width:2px,color:#fff
    classDef cd fill:#ef233c,stroke:#d90429,stroke-width:2px,color:#fff
    classDef aws fill:#ffb703,stroke:#fb8500,stroke-width:2px,color:#000
    
    Dev((Developer)) --> PR(GitHub PR):::git
    
    subgraph CI ["Continuous Integration"]
        direction TB
        Lint(Lint & Unit Test):::ci
        Contract(Contract Tests):::ci
        Scan(Amazon Inspector / Trivy):::ci
        Build(Docker Build):::ci
        
        Lint --> Contract --> Scan --> Build
    end
    
    PR --> CI
    
    Build -- "Image Push" --> ECR(Amazon ECR):::aws
    
    subgraph CD ["Continuous Deployment"]
        direction TB
        CDK(AWS CDK - IaC Deploy):::cd
        Staging(Staging ECS Rollout):::cd
        Gate{Automated<br/>Integration Tests}:::cd
        ProdGate{Manual<br/>Approval}:::cd
        Prod(Production - Blue/Green):::aws
        
        CDK --> Staging --> Gate --> ProdGate --> Prod
    end
    
    ECR --> CD
```

### Well-Architected Pillars Covered Here
* **Operational Excellence:** Infrastructure as Code (CDK) with automated Blue/Green rollbacks for safe deployments.
* **Security:** Automated Trivy/Inspector container scanning blocks vulnerabilities before production.
* **Reliability:** Pre-deployment contract tests guarantee safe inter-service communication.

---

## 🔭 3. Observability & Async Messaging (Operations)

```mermaid
flowchart TD
    classDef compute fill:#ade8f4,stroke:#0077b6,stroke-width:2px,color:#000
    classDef msg fill:#cdb4db,stroke:#9c89b8,stroke-width:2px,color:#000
    classDef obs fill:#ffc8dd,stroke:#ffafcc,stroke-width:2px,color:#000
    classDef fin fill:#e9c46a,stroke:#e76f51,stroke-width:2px,color:#000
    
    Services[Microservices<br/>ECS & Lambda]:::compute
    
    subgraph Logging & Alerting ["Unified Observability Stack"]
        Firehose(Kinesis Data Firehose<br/>PII Masking):::compute
        ELK[(OpenSearch / Elastic)]:::fin
        CW(Amazon CloudWatch<br/>Logs & Metrics):::obs
        SNS(Amazon SNS - Alert Router):::msg
        
        Services -- "Raw Logs" --> Firehose
        Firehose -- "Sanitized Logs" --> ELK
        Services -- "Metrics (Latency/Errors)" --> CW
        CW -- "Alarms (> 5xx threshold)" --> SNS
        SNS -- "Page / Triage" --> Ops((On-call Engineer))
    end
    
    subgraph Event-Driven Architecture ["Async Business Flow"]
        EB(EventBridge - Event Bus):::msg
        Target1[Notification Engine]:::compute
        Target2[Tax Calculator]:::compute
        
        Services -- "Domain Events<br/>e.g. DepositSettled" --> EB
        EB -- "Pub/Sub Fanout" --> Target1
        EB -- "Pub/Sub Fanout" --> Target2
    end
```

### Well-Architected Pillars Covered Here
* **Operational Excellence:** Centralized CloudWatch alarms and SNS routing enable proactive incident response.
* **Security:** Kinesis Data Firehose actively masks plaintext PII before it reaches searchable log indices.
* **Cost Optimization & Sustainability:** Event-driven pub/sub removes polling waste, reducing carbon footprint and baseline costs.
