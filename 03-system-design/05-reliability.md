# 3 Core Ideas for a Reliable AWS Architecture
 
 To make sure Alborz Bank never goes offline and never loses customer data, we follow three simple rules on AWS:
 
 ### 1. Multi-AZ Deployments (Never Rely on One Building)
 We distribute our application servers and databases (like Amazon Aurora) across 3 entirely separate physical data centers in a region (called Availability Zones or "AZs"). If one data center experiences a total power outage, the Load Balancer instantly forwards traffic to the remaining healthy buildings. The customers never notice a glitch.
 
 ### 2. Auto-Scaling (Elastic Growth)
 On payday (the 25th of the month), traffic drastically spikes. Instead of running just a few fixed servers that might crash under the massive load, we use AWS Auto-Scaling. The system automatically detects the traffic spike, spins up 20 extra servers to handle the rush, and seamlessly turns them back off when traffic cools down.
 
 ### 3. Dead Letter Queues (Never Lose a Message)
 If our Onboarding service tries to send an event to the Core Ledger, but the Ledger is temporarily unhealthy, the message isn't lost. We use AWS EventBridge and SQS to catch the failure and automatically retry the action 5 minutes later. If it fails repeatedly, the message is safely stored in a "Dead Letter Queue" (DLQ) where an engineer is alerted, fixes the bug, and replays the message manually without losing any customer money!
 
 ---
 
 ## Visualizing the Reliability Flow
 
 ```mermaid
 graph TD
     classDef client fill:#1E293B,stroke:#0F172A,stroke-width:2px,color:#FFFFFF,rx:8px,ry:8px;
     classDef aws fill:#F97316,stroke:#9A3412,stroke-width:2px,color:#FFFFFF,rx:8px,ry:8px;
     classDef az fill:#F1F5F9,stroke:#94A3B8,stroke-width:2px,color:#0F172A,rx:8px,ry:8px;
     classDef db fill:#0284C7,stroke:#0369A1,stroke-width:2px,color:#FFFFFF,rx:8px,ry:8px;
     classDef queue fill:#FEE2E2,stroke:#DC2626,stroke-width:2px,color:#7F1D1D,rx:8px,ry:8px;
 
     User([Customer App]):::client --> ALB[AWS Application Load Balancer]:::aws
     
     subgraph autoScaling [Idea 2: Auto-Scaling]
     subgraph multiAZ [Idea 1: Multi-AZ Data Centers]
         ALB --> S1[Bank Service Server AZ 1]:::az
         ALB --> S2[Bank Service Server AZ 2]:::az
         ALB -. "Auto-spins up if Traffic is High" .-> S3[Rescue Server AZ 3]:::az
     end
     end
     
     subgraph multiAZDB [Multi-AZ Database Engine]
         S1 --> DB_W[(Aurora Primary DB)]:::db
         S2 --> DB_R[(Aurora Backup DB)]:::db
         DB_W -. "Live Data Replication" .-> DB_R
     end
 
     subgraph idea3 [Idea 3: Retries & Dead Letter Queues]
         S1 -- "Async Message" --> EventBus{EventBridge Bus}:::aws
         EventBus -- "Retries on Failure" --> S2
         EventBus -. "After 5 Fails, move to" .-> DLQ(Dead Letter Queue):::queue
     end
 ```
