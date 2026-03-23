# AWS Tagging & Cost Optimization Strategy (FinOps)
 
 Effective AWS cost management for Alborz Bank requires a shift from reactive billing to proactive automated governance. This document outlines the explicit tagging taxonomy and architectural rules to keep our microservices cost-efficient.
 
 ---
 
 ## 1. The FinOps Continuous Loop
 
 Cost optimization is not a one-time event; it is a continuous feedback loop driven by visibility and automated governance.
 
 ```mermaid
 flowchart LR
     classDef visibility fill:#e0fbfc,stroke:#3d5a80,stroke-width:2px,color:#000
     classDef usage fill:#eece1a,stroke:#d4a373,stroke-width:2px,color:#000
     classDef rates fill:#e9edc9,stroke:#606c38,stroke-width:2px,color:#000
     classDef gov fill:#ffd6ff,stroke:#c47aff,stroke-width:2px,color:#000
 
     Visibility["1. Visibility & Allocation<br/>(Tagging & SCPs)"]:::visibility --> Usage
     
     Usage("2. Usage Optimization<br/>Rightsizing & Dev Scheduling"):::usage --> Rates
     
     Rates("3. Rate Optimization<br/>Spot, Graviton, Savings Plans"):::rates --> Gov
     
     Gov{"4. FinOps Governance<br/>Budgets & Anomaly Alerts"}:::gov -.->|Iterate| Visibility
 ```
 
 ---
 
 ## 2. Goldilocks Tagging Taxonomy
 
 To achieve 100% cost allocation tracking inside the AWS Billing Console, we enforce a strict 4-tag minimum across all resources using **AWS Organizations Service Control Policies (SCPs)**.
> If an engineer attempts to deploy an RDS instance or ECS cluster without these tags, the deployment will fail.
 
 | Tag Key | Description | Allowed Values (Alborz Bank Context) |
 | :--- | :--- | :--- |
 | `Environment` | Separation of billing by deployment stage. | `Dev`, `Staging`, `Prod` |
 | `Owner` | The engineering squad responsible for the service. | `Team1-Onboarding`, `Team2-Deposits`, `Team3-Payments`, `Team4-Platform` |
 | `CostCenter` | Finance tracking identifier for chargebacks. | `CC-100-ONB`, `CC-200-DEP`, `CC-300-PAY`, `CC-400-PLAT` |
 | `ApplicationID` | Granular grouping for microservice boundaries. | `App-Auth`, `App-CoreLedger`, `App-Payouts`, `App-Registration` |
 
 > **⚠️ Critical:** Tags must be activated manually in the AWS *Cost Allocation Tags* console before they appear in Cost Explorer.
 
 ---
 
 ## 3. Architectural Cost Optimization Rules
 
 We map standard FinOps best practices directly to the Alborz Bank service topology.
 
 | Action Pillar | Optimization Rule | Application within Alborz Bank Architecture |
 | :--- | :--- | :--- |
 | **Compute (Rates)** | **Migrate to AWS Graviton** | Switch ECS Fargate tasks and AWS Lambdas (e.g., *Payout Orchestrator*, *Auth API*) from x86 to Graviton (ARM64) for up to 40% better price-performance. |
 | **Compute (Usage)** | **Schedule Non-Prod Workloads** | Use AWS Instance Scheduler to automatically spin down `Dev` and `Staging` environments (Aurora DBs, ECS Tasks) from 7 PM to 8 AM and weekends. Conserves ~70% of compute costs. |
 | **Compute (Rates)** | **Leverage Spot Instances** | Use EC2 Spot Instances for fault-tolerant, stateless workloads like our CI/CD GitHub Actions runners or background asynchronous CAMT file parsers (Up to 90% savings). |
 | **Storage (Usage)** | **Automate S3 Lifecycles** | Configure the `Regulatory S3 Archive` (Payments Team) to transition old CAMT clearing files to **S3 Standard-IA** after 30 days, and **Glacier Deep Archive** after 1 year. |
 | **Network (Usage)** | **Minimize Data Transfer Costs** | Keep microservices interacting with shared DBs inside the same Availability Zone. Route all `App-Auth` traffic to DynamoDB via a **VPC Endpoint**, entirely bypassing expensive NAT Gateway $0.045/GB fees. |
 | **Governance** | **Auto-Alerting & Rightsizing** | Enable **AWS Cost Anomaly Detection** to alert Slack if a Lambda loop occurs. Review **AWS Compute Optimizer** bi-weekly to downsize over-provisioned Aurora PostgreSQL instances. |
 
 ---
 
 ## 4. Where Our Savings Come From
 
 By enforcing the above rules, cloud expenditure can be drastically reduced before resorting to long-term commitments like 3-Year Compute Savings Plans.

>Note that these numbers are not gauranteed, for an actual projection you need to use AWS Cost Explorer and put more details about your workloads to make sure you are making the right decisions.

 ```mermaid
 pie title Estimated Impact on Cloud Bill
     "Spot Instances (CI/CD, Batch Workers)" : 30
     "Dev/Test Auto-Shutdown (Instance Scheduler)" : 25
     "Graviton Architecture Migration" : 20
     "S3 Lifecycle Transitions" : 15
     "Rightsizing & NAT Gateway Bypasses" : 10
 ```
 
 ---
 
 **Next Steps:** Form a lightweight Cloud Center of Excellence (CoE) meeting monthly to review Cost Explorer against the `CostCenter` / `Owner` tags to hold each of the 4 teams accountable for their microservice footprint.


<br><br><br>

## Note: My ideas about cost optimization comes from a book.
I am recently reading a book called “MASTERING AWS COST OPTIMIZATION” by Eli Mansoor & Yair Green. It is a great book and I highly recommend it to anyone who wants to learn more about AWS cost optimization.