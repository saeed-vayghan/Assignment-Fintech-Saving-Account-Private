# Performance and Cost Efficiency
 
 When designing a cloud-native architecture for a multi-domain business like ours, the balance between **Performance** and **Cost Efficiency** is a continuous exercise. 
 
 This document explores a recent, high-profile architectural shift in the industry purely as a starting point for evaluating our own workloads. It is **not** a mandate that we abandon serverless or immediately start rebuilding monoliths. Instead, it highlights why we must dynamically monitor the relationship between data transfer and billing models at scale.
 
 ---
 
 ## 1. Case Study Exploration: Amazon Prime Video Quality Analysis
 
 Recently, Amazon's Prime Video Quality Analysis team shifted a critical monitoring service from a highly distributed Serverless architecture (AWS Lambda + Step Functions) back to a single Monolith (ECS on EC2). 
 
 ### Why the Shift?
 In their original Serverless design, every second of a video stream processing triggered a state transition in AWS Step Functions. Because Serverless bills *per-invocation* and *per-state-transition*, the costs grew extremely high under their relentless, massive scale. Furthermore, transferring heavy payloads between distributed Lambda components required reading and writing to centralized storage (S3 Buckets), incurring massive network and data transfer fees.
 
 ### How Moving to a Monolith Reduced Costs
 By refactoring the disparate Serverless workflow into a single cohesive process hosted on standard EC2 compute instances (via ECS), Amazon achieved an incredible **90% reduction in infrastructure costs**!
 
 1. **Eliminated State Transition Costs:** They were no longer billed for thousands of micro-steps by Step Functions. The entire workflow ran freely inside the application's RAM.
 2. **Eliminated Cross-Component Data Transfer:** Because all logic executed within the exact same container, the process didn't need to serialize data over the network or pay to upload/download state from S3.
 
 ---
 
 ## 2. Takeaways for Alborz Bank
 
 This Amazon case study is an excellent reminder that **"Serverless is infinitely scalable, but not necessarily the most cost-effective at a constant, aggressive baseline scale."** 
 
 However, we must stress that **we are not adopting this as a blanket rule to return to monolithic design.** This is just a sample exploration to provoke critical thinking.
 
 For our own architecture, we need significantly more deep work and profiling on our specific issues before making structural changes. We will apply the following guidelines moving forward:
 
 - **Profile the Billing Model First:** We must regularly analyze if a service experiences erratic, unpredictable bursts (where Serverless shines) or a constant, predictable, high-throughput baseline (where provisioned EC2/Containers are often vastly cheaper).
 - **Beware the "Network Tax":** If two of our microservices are constantly passing massive data loads between them synchronously, we may need to reconsider component boundaries to avoid serialization overhead.
 - **Right-Size by Domain:** We will continue to evaluate the cost-to-performance ratio independently across our teams (e.g., Onboarding vs. Core Ledger) rather than forcing a single architectural pattern bank-wide.
