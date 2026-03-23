# Alborz Bank REST API Guidelines
 
 To ensure our microservices across all teams (Onboarding, Deposits, Payments, Platform) remain scalable, maintainable, reliable, and secure, every REST API must adhere to the following global standards.
 
 ## 1. Scalability & Performance
 
 - **Statelessness:** APIs must strictly be stateless. No client context should be stored on the server between requests. All necessary context must be provided in the request (e.g., via auth tokens).
 - **Pagination:** Any endpoint returning a collection (e.g., transaction history) **must** support pagination. Use `limit` and `cursor` (or `offset` if cursor is not feasible) as query parameters.
 - **Filtering & Sorting:** Use URL query parameters for filtering (`?status=ACTIVE`) and sorting (`?sort=-createdAt` for descending).
 - **Caching:** Use appropriate HTTP Cache headers (`ETag`, `Cache-Control`) for static or infrequently changing resources (e.g., Legal PDFs, Interest Rates).
 
 ## 2. Maintainability & Consistency
 
 - **Resource-Oriented URLs:** Endpoints must represent nouns, not verbs. Use plural nouns.
   - ✅ `POST /accounts`
   - ❌ `POST /createAccount`
 - **URI Case:** URLs must be in kebab-case (`/fixed-term-deposits`). 
 - **JSON Payloads:** JSON payloads must be in camelCase (`{ "accountId": "123" }`).
 - **Versioning:** Version APIs at the URI level to gracefully handle breaking changes (e.g., `v1/customers`).
 - **Documentation:** Every service must maintain an OpenAPI 3.0 (Swagger) specification. The spec is the contract.
 - **Standardized Error Responses:** Errors must follow a consistent JSON structure (e.g., RFC 7807 Problem Details).
   ```json
   {
     "error": "INSUFFICIENT_FUNDS",
     "message": "The account balance is too low for this withdrawal.",
     "correlationId": "req-123456"
   }
   ```
 
 ## 3. Reliability & Fault Tolerance
 
 - **Idempotency:** All mutating requests (`POST`, `PUT`, `PATCH`) that deal with financial actions MUST include an `Idempotency-Key` header.
 - **Correlation IDs:** Every incoming API request at the Edge/API Gateway must be assigned an `X-Correlation-ID`. This ID must be forwarded to all downstream microservices and logged.
 - **Appropriate HTTP Methods:**
   - `GET` - Read (Idempotent, Safe)
   - `POST` - Create (Not Idempotent)
   - `PUT` - Complete Update / Replace (Idempotent)
   - `PATCH` - Partial Update (Not strictly Idempotent)
   - `DELETE` - Remove (Idempotent)
 - **Appropriate Status Codes:** Standardize around core codes:
    - `200 OK`
    - `201 Created`
    - `204 No Content`
    - `400 Bad Request`
    - `401 Unauthorized`
    - `403 Forbidden`
    - `404 Not Found`
    - `409 Conflict` (e.g., state machine violations)
    - `429 Too Many Requests`
    - `500 Server Error`
 
 ## 4. Security & Compliance
 
 - **Authentication:** All private endpoints must require a valid Bearer token.
 - **Authorization (Zero Trust):** The API Gateway extracts the `userId` from the token. The downstream service must mathematically enforce that the `userId` owns the `accountId` being requested.
 - **Data Masking in URLs:** Never place sensitive PII (like a National ID / PNR or strict Account Numbers) in the URL path. Send sensitive identifiers in the JSON body or use UUIDs in paths.
 - **Rate Limiting:** Protect all endpoints using API Gateway usage plans and AWS WAF rate-limiting rules to prevent DDoS or enumeration attacks.
 - **CORS Policies:** Cross-Origin Resource Sharing must be strictly locked down to trusted Web App origins.


<br><br><br>

# Note:

This document is just a starting point to explain the importance and the rules of API design.

In and actual Production environment, we would have more rules and guidelines to follow.

I would suggest to have an:

- Internal documentation for API design and development standards.
- AI based tool or MCP server to audit and validate API designs against these standards.