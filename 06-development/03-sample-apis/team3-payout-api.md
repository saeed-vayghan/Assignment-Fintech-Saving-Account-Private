# Team 3: Payout API (Payments & Accounting)
 
 This API is responsible for initiating outbound fiat clearing instructions, tracking payout lifecycles, and ensuring absolute idempotency against network retries.
 
 ## Exposed REST APIs
 - `POST /v1/payouts` - Manually trigger a payout to an external verified bank account.
 - `GET /v1/payouts/{payoutId}` - Check the real-time status of a clearing job.
 
 ## OpenAPI / Swagger Definition
 Copy the following YAML and paste it into [editor.swagger.io](https://editor.swagger.io/) to view the UI.
 
 ```yaml
 openapi: 3.0.3
 info:
   title: Payout API (Team 3)
   version: 1.0.0
   description: Orchestrates outbound SEPA/SWIFT clearing batches.
 servers:
   - url: https://api.alborzbank.com
 paths:
   /v1/payouts:
     post:
       summary: Initiate a new fiat payout
       parameters:
         - name: X-Correlation-ID
           in: header
           required: true
           schema:
             type: string
             example: "req-abc-123"
       requestBody:
         required: true
         content:
           application/json:
             schema:
               type: object
               required:
                 - accountId
                 - targetIban
                 - amount
                 - currency
               properties:
                 accountId:
                   type: string
                   example: "acc_112233"
                 targetIban:
                   type: string
                   example: "SE12345678901234567890"
                 amount:
                   type: number
                   example: 500.00
                 currency:
                   type: string
                   example: "SEK"
       responses:
         '202':
           description: Payout Accepted and Enqueued.
           content:
             application/json:
               schema:
                 type: object
                 properties:
                   payoutId:
                     type: string
                     example: "pay_12345abc"
                   status:
                     type: string
                     example: "QUEUED"
 
   /v1/payouts/{payoutId}:
     get:
       summary: Get payout job status
       parameters:
         - name: X-Correlation-ID
           in: header
           required: true
           schema:
             type: string
             example: "req-abc-123"
         - name: payoutId
           in: path
           required: true
           schema:
             type: string
       responses:
         '200':
           description: Status of the payout job.
           content:
             application/json:
               schema:
                 type: object
                 properties:
                   payoutId:
                     type: string
                   status:
                     type: string
                     example: "SUBMITTED_TO_CLEARING"
 ```
 
 ## Sample cURL Requests
 
 **1. Initiate a Payout**
 ```bash
 curl -X POST "https://api.alborzbank.com/v1/payouts" \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer <opaque_token>" \
      -H "X-Correlation-ID: req-abc-123" \
      -H "Idempotency-Key: pay-retry-987" \
      -d '{
            "accountId": "acc_112233",
            "targetIban": "SE12345678901234567890",
            "amount": 500.00,
            "currency": "SEK"
          }'
 ```
 
 **2. Check Payout Status**
 ```bash
 curl -X GET "https://api.alborzbank.com/v1/payouts/pay_12345abc" \
      -H "Authorization: Bearer <opaque_token>" \
      -H "X-Correlation-ID: req-abc-123"
 ```
