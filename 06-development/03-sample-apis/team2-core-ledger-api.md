# Team 2: Core Ledger API (Deposits & Transactions)
 
 This API strictly manages the financial ledgers, maintaining mathematical ACID compliance for all deposit and withdrawal events.
 
 ## Exposed REST APIs
 - `GET /v1/accounts/{accountId}/balance` - Retrieve the realtime calculated balance.
 - `GET /v1/accounts/{accountId}/transactions` - Get paginated transaction history.
 - `POST /v1/accounts/{accountId}/withdrawals` - Initiate a PYO (Pay-Out) withdrawal request.
 
 ## OpenAPI / Swagger Definition
 Copy the following YAML and paste it into [editor.swagger.io](https://editor.swagger.io/) to view the UI.
 
 ```yaml
 openapi: 3.0.3
 info:
   title: Core Ledger API
   version: 1.0.0
   description: Manages strictly ACID compliant financial ledgers.
 servers:
   - url: https://api.alborzbank.com
 paths:
   /v1/accounts/{accountId}/withdrawals:
     post:
       summary: Request a withdrawal (PYO)
       parameters:
         - name: X-Correlation-ID
           in: header
           required: true
           schema:
             type: string
             example: "req-abc-123"
         - name: accountId
           in: path
           required: true
           schema:
             type: string
       requestBody:
         required: true
         content:
           application/json:
             schema:
               type: object
               required:
                 - amount
                 - currency
               properties:
                 amount:
                   type: number
                   example: 500.00
                 currency:
                   type: string
                   example: "SEK"
       responses:
         '201':
           description: Withdrawal initiated.
           content:
             application/json:
               schema:
                 type: object
                 properties:
                   transactionId:
                     type: string
                     example: "txn_555666"
                   status:
                     type: string
                     example: "PROCESSING"
         '409':
           description: Insufficient funds.
 
   /v1/accounts/{accountId}/balance:
     get:
       summary: Real-time balance
       parameters:
         - name: X-Correlation-ID
           in: header
           required: true
           schema:
             type: string
             example: "req-abc-123"
         - name: accountId
           in: path
           required: true
           schema:
             type: string
       responses:
         '200':
           description: Current balance.
           content:
             application/json:
               schema:
                 type: object
                 properties:
                   availableBalance:
                     type: number
                     example: 10500.00
                   currency:
                     type: string
                     example: "SEK"
 ```
 
 ## Sample cURL Requests
 
 **1. Request a Withdrawal**
 ```bash
 curl -X POST "https://api.alborzbank.com/v1/accounts/acc_112233/withdrawals" \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer <opaque_token>" \
      -H "X-Correlation-ID: req-abc-123" \
      -H "Idempotency-Key: uuid-456-def" \
      -d '{
            "amount": 500.00,
            "currency": "SEK"
          }'
 ```
 
 **2. Retrieve Balance**
 ```bash
 curl -X GET "https://api.alborzbank.com/v1/accounts/acc_112233/balance" \
      -H "Authorization: Bearer <opaque_token>" \
      -H "X-Correlation-ID: req-abc-123"
 ```
