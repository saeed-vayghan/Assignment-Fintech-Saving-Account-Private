# Onboarding Team: Architecture Explanation & Nuances

This document explains the architectural decisions behind dividing the Onboarding team's domain into the four specific microservices exactly as defined in the `01-onboarding.md` profile.

### The 4 Microservices:
1. `Registration & Eligibility API`
2. `KYC & IDV Webhook Service`
3. `Compliance & AML Service`
4. `Bank Validation Service`
5. `Underwriting & Decision Engine`
6. `Legal Document API`

---

## 1. Are they Serverless?
**Yes, absolutely.** As defined in the `04-teams.md` document, the compute layer for the Onboarding team is strictly **AWS Lambda** and **Amazon DynamoDB**.

In a serverless context, these "microservices" are typically implemented as independent groups of Lambda functions sitting behind distinct AWS API Gateway routes. This allows them to scale to zero when unused and scale infinitely during sudden traffic spikes without provisioning or paying for idle servers.

---

## 2. Why exactly these 6 boundaries? (The Challenge)

Let's challenge if we have too many or too few services by evaluating their Domain-Driven Design (DDD) boundaries and failure domains.

### 1. Registration & Eligibility API
* **Why it exists:** This is the synchronous front-door for the Customer Web App. When a user clicks "Next" on a form, they expect an immediate response (e.g., "Age requirement failed").
* **Challenge:** *Could we put everything in here?* No. If we tightly coupled third-party API calls (like open banking or ID verification) directly into this synchronous flow, any vendor downtime would completely crash the user's registration experience. It must stay lightweight and purely handle state-machine transitions and fast database reads/writes.

### 2. KYC & IDV Webhook Service
* **Why it exists:** Vendors like Onfido or Signicat process identity documents asynchronously. They send an HTTP POST (webhook) to the system minutes or hours later.
* **Challenge:** *Why not merge this with Registration?* Decoupling webhooks is a serverless best practice. If an IDV vendor suddenly sends a massive batch of 10,000 delayed webhooks at once, a dedicated Lambda function will seamlessly digest them. If this was merged into the Registration API, that sudden webhook spike could exhaust the database connection pool or AWS concurrency limits, preventing actual humans from signing up.

### 3. Compliance & AML Service
* **Why it exists:** This handles PEP/Sanctions screening, reads massive amounts of transaction data for AML monitoring, and serves the internal **Compliance Officer Dashboard**.
* **Challenge:** *Why separate it?* **Security and RBAC.** The Registration API is exposed to the public internet using low-privilege temporary session tokens (or federated eIDs like BankID). The Compliance Service, however, exposes highly sensitive PII and the ability to manually override account states. By isolating it into its own service, we can apply much stricter AWS IAM execution roles, deploy it behind internal-only subnets, and tightly enforce the `Compliance_Officer` Auth-Service tokens.

### 4. Bank Validation Service (Plaid/Tink)
* **Why it exists:** Open Banking flows involve redirecting the user to their own bank, doing OAuth handshakes, and returning.
* **Challenge:** *Why not merge this with Registration since it happens during onboarding?* Reusability. Right now, a customer validates an IBAN during onboarding. However, 6 months from now (in Phase 2 or V2), a customer might want to add a *second* bank account to withdraw funds to. If the Bank Validation logic is tightly coupled to the Onboarding Registration state machine, it will be very difficult to reuse. As an independent service, the Deposits team can easily reuse it later for new IBAN validations.

### 5. Underwriting & Decision Engine (AWS Step Functions)
* **Why it exists:** This engine performs the final evaluation of all compliance inputs (KYC, PEP, IDV) and outputs a mathematically verifiable `APPROVED` or `REJECTED` state.
* **Challenge:** *Why not evaluate these rules directly inside the Registration API?* Decoupling the rules engine allows the Risk & Compliance team to update their underwriting criteria (e.g., changing PEP tolerance) independently, without a redeployment of the user-facing Registration API. Furthermore, modeling this as an **AWS Step Function** provides a flawless, visual, and immutable audit trace of exactly which decision branch a customer flowed down, which is a massive regulatory compliance benefit.

### 6. Legal Document API
* **Why it exists:** To act as the single source of truth for all customer-facing legal agreements (T&C, Privacy Policy, Fee Schedules) during the signup flow.
* **Challenge:** *Why not just serve static PDFs from a public CDN?* **Auditability.** In banking, proving *exactly* which version of a Terms & Conditions document a user signed at a specific timestamp (e.g., `v1.2` vs `v1.3`) is a strict legal requirement. By placing an API in front of the S3 bucket, the API dynamically fetches the exact semantic version of the PDF required for that user's region, and guarantees the exact `S3 Version ID` is immutably embedded into their DynamoDB application record.

---

## Conclusion
The 4-microservice split is perfectly tuned for a **Serverless ecosystem**. It prevents a cascading failure (e.g., Tink going down won't stop a user from filling out their initial registration form), it isolates highly sensitive back-office endpoints from public-facing web routes, and it safely separates unpredictable asynchronous webhooks from synchronous human interactions.
