# CI/CD and Deployment Strategy
 
 This document defines the automated path from a developer's local machine to the production cloud environments for Alborz Bank microservices, ensuring zero-downtime, safe, and highly observable releases.
 
 ---
 
 ## 1. The Three Environments
 
 To maintain absolute stability, we utilize three strictly segregated deployment environments:
 
 1. **Playground:** Developer sandbox environment.
 2. **Staging:** An exact replica of the production environment.
 3. **Production:** The live, customer-facing environment.
 
 ---
 
 ## 2. The Development & Deployment Flow

 ```mermaid
 flowchart TD
     classDef local fill:#e9ecef,stroke:#6c757d,stroke-width:2px,color:#000
     classDef ci fill:#fff3b0,stroke:#e9c46a,stroke-width:2px,color:#000
     classDef env fill:#ade8f4,stroke:#0077b6,stroke-width:2px,color:#000
     classDef prod fill:#e2ece9,stroke:#2a9d8f,stroke-width:2px,color:#000
 
     subgraph "Phase 1: Local & CI"
         L[🏠 1. Developer Runs Local Tests]:::local --> C[💻 2. Push to Feature Branch]:::local
         C --> CI[⚙️ 3. CI Pipeline: Test & Build]:::ci
         CI -- "Docker Image to ECR" --> ECR[(Artifact Registry)]
     end
 
     subgraph "Phase 2: Staging & QA"
         ECR -- "Pull Request Opened" --> STG[🧪 4. CD: Deploy to Staging]:::env
         STG --> QA[🕵️ 5. QA & Engineers Test]:::local
         QA -- "Approve PR" --> Merge[🔀 Merge to Master]:::ci
     end
 
     subgraph "Phase 3: Production & Playground"
         Merge --> CD_Prod[🚀 6. CD: Deploy to Production]:::prod
         Merge -. "Optional Parallel Deploy" .-> PG[🎮 Deploy to Playground]:::env
         CD_Prod --> Canary[🐦 7. Canary Deployment Strategy]:::prod
         Canary --> Health[🩺 8. Application Healthchecks]:::prod
         Health -- "If 100% Healthy" --> Drain[♻️ 9. Terminate Old Instances]:::prod
         Health -- "If Failing" --> Rollback[⏪ Auto-Rollback]:::prod
     end
 
     subgraph "Phase 4: Global Observability"
         Drain --> Obs[📊 10. ELK / Datadog Aggregation]:::local
     end
 ```
 
 ---
 
 ## 3. Deep Dive: The Canary Deployment Lifecycle
 
 We strictly utilize a **Canary Deployment Strategy** in Production to minimize blast radius. Instead of swapping all containers simultaneously, we slowly shift traffic.
 
 Every microservice container exposes a `/health` REST API endpoint. The container orchestration (e.g., AWS ECS or Kubernetes) continuously polls this endpoint to verify readiness and liveness.
 
 ```mermaid
 sequenceDiagram
     autonumber
     
     participant CD as CI/CD Orchestrator
     participant LB as Load Balancer
     participant V1 as Service (v1.0.0)
     participant V2 as Service (v2.0.0-Canary)
     participant Telemetry as Datadog / ELK
 
     Note over CD, Telemetry: Initiation
     CD->>V2: Spin up 10% of total Fleet (v2.0.0)
     
     loop Polling Configuration
         CD->>V2: GET /health
         V2-->>CD: 200 OK (DB Connected, Memory Good)
     end
     
     Note over CD, Telemetry: Canary Traffic Shift
     CD->>LB: Route 5% of traffic to Canary (v2.0.0)
     
     par Live Telemetry Monitoring
         V1-->>Telemetry: Stream Metrics, Traces, Logs (version: v1.0.0)
         V2-->>Telemetry: Stream Metrics, Traces, Logs (version: v2.0.0)
     end
     
     Telemetry->>CD: No elevated error rates detected on v2.0.0
     
     Note over CD, Telemetry: Full Cutover & Cleanup
     CD->>LB: Shift 100% of traffic to v2.0.0
     CD->>V2: Spin up remaining 90% of Fleet
     CD->>V1: Send SIGTERM to old instances
     V1->>V1: Drain active connections & Terminate
     
     CD->>Telemetry: Annotate Dashboard: "Deployed v2.0.0"
 ```
 
 ---
 
 ## 4. Checklist & Requirements
 
 - **Healthchecks:** Every single web server, backend service, and async worker MUST expose a `/health` endpoint.
 - **Graceful Termination:** Services must handle `SIGTERM` signals correctly to ensure zero dropped requests when old instances are terminated.
 - **Unified Observability:** Every log must be tagged with the specific `version-id`.
