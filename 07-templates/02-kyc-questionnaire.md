# Template: KYC (Know Your Customer) Questionnaire
 
 **Service Dependency:** Team 1 (Onboarding API)
 **Purpose:** Standardize the JSON payload for the pre-fill KYC application process.
 
 ---
 
 ## Payload Schema Definition
 
 When a client submits their KYC answers via `PATCH /v1/applications/{id}`, the payload must adhere to this 5-question schema, containing both `SELECT` and `FREE_TEXT` variable types as per compliance requirements.
 
 ```json
 {
   "questionnaire": [
     {
       "questionId": "q1_source_of_funds",
       "type": "SELECT",
       "questionText": "What is the primary source of the funds you intend to deposit?",
       "allowedOptions": ["Salary", "Savings", "Inheritance", "Sale of Property", "Other"],
       "answer": "Salary"
     },
     {
       "questionId": "q2_annual_volume",
       "type": "SELECT",
       "questionText": "What is your estimated annual deposit volume into this saving account?",
       "allowedOptions": ["< €10,000", "€10,000 - €50,000", "€50,000 - €100,000", "> €100,000"],
       "answer": "€10,000 - €50,000"
     },
     {
       "questionId": "q3_occupation",
       "type": "SELECT",
       "questionText": "What is your primary occupation/employment status?",
       "allowedOptions": ["Employed", "Self-Employed", "Student", "Retired", "Unemployed"],
       "answer": "Employed"
     },
     {
       "questionId": "q4_pep_status",
       "type": "SELECT",
       "questionText": "Are you, or is any immediate family member, a Politically Exposed Person (PEP)?",
       "allowedOptions": ["Yes", "No"],
       "answer": "No"
     },
     {
       "questionId": "q5_additional_context",
       "type": "FREE_TEXT",
       "questionText": "Please provide any additional context regarding your intended account usage (Optional):",
       "answer": "I intend to use this account to save for a down payment on a house over the next 3 years."
     }
   ]
 }
 ```
 
 ---
 
 ## Validation Rules (Backend)
 1. **Data Completeness:** The `answer` field is strictly required for questions `q1` through `q4`. Question `q5` is optional.
 2. **Enum Enforcement:** If `type` == `SELECT`, the backend must strictly validate that the provided `answer` string matches an exact string in the `allowedOptions` array. If it doesn't, return a `400 Bad Request`.
 3. **PEP Escalation:** If `q4_pep_status` == `Yes`, the onboarding application status should automatically route to `MANUAL_REVIEW` instead of `APPROVED` to trigger a human compliance check.
