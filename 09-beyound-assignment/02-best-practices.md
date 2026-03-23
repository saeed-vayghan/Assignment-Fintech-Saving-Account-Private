# Bonus Best Practices
 
 This document outlines the core engineering and operational standards applied across all cross-functional teams to ensure autonomy, security, and consistency.
 
 ### 1. Architecture & Design
 - **Resource Naming:** Standardized cloud identifiers (e.g., `Qrn:<Namespace id - NID>:<Namespace-specific string - NSS>`).
 - **Messaging Compatibility:** Enforce backward and forward schema compatibility across all async EventBus payloads.
 - **Availability Classes:** Define strict tier-based uptime and SLA classes per domain service.
 
 ### 2. Security & Compliance
 - **Access Management:** Enforce least-privilege AWS IAM roles, resource policies, and Zero-Trust API authorization.
 - **Vulnerability Management:** Continuous automated code/container scanning blocking CI on Critical CVEs.
 
 ### 3. Operations & Reliability
 - **Disaster Recovery:** Automated multi-AZ failover plans and routine architectural resilience testing.
 - **Feature Flags:** Utilize an experimentation platform to safely decouple infrastructure deployments from business feature rollouts.
 
 ### 4. Engineering Standards
 - **Service Documentation:** Mandatory architectural blueprints, living ADRs, and OpenAPI interface definitions.
