# Team 1: Registration API (Onboarding & Compliance)
 
 This API handles the customer onboarding journey, managing the eligibility, draft, and activation states of a new user.
 
 ## Exposed REST APIs
 - `POST /v1/applications` - Start a new onboarding application.
 - `GET /v1/applications/{applicationId}` - Retrieve the current status of an application.
 - `PATCH /v1/applications/{applicationId}` - Submit KYC questionnaire answers.
 
 ## OpenAPI / Swagger Definition
 Copy the following YAML and paste it into [editor.swagger.io](https://editor.swagger.io/) to view the UI.
 
 ```yaml
 openapi: 3.0.3
 info:
   title: Onboarding Registration API
   version: 1.0.0
   description: Manages customer onboarding applications.
 servers:
   - url: https://api.alborzbank.com
 paths:
   /v1/applications:
     post:
       summary: Create a new application
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
                 - legalName
                 - dateOfBirth
                 - nationality
               properties:
                 legalName:
                   type: string
                   example: "John Doe"
                 dateOfBirth:
                   type: string
                   format: date
                   example: "1990-05-15"
                 nationality:
                   type: string
                   example: "SE"
       responses:
         '201':
           description: Application created in DRAFT state.
           content:
             application/json:
               schema:
                 type: object
                 properties:
                   id:
                     type: string
                     example: "app_987654321"
                   status:
                     type: string
                     example: "DRAFT"
 
   /v1/applications/{applicationId}:
     get:
       summary: Get application status
       parameters:
         - name: X-Correlation-ID
           in: header
           required: true
           schema:
             type: string
             example: "req-abc-123"
         - name: applicationId
           in: path
           required: true
           schema:
             type: string
       responses:
         '200':
           description: Current state of the application.
           content:
             application/json:
               schema:
                 type: object
                 properties:
                   id:
                     type: string
                   status:
                     type: string
                     example: "PENDING_IDV"
 ```
 
 ## Sample cURL Requests
 
 **1. Create a new Application**
 ```bash
 curl -X POST "https://api.alborzbank.com/v1/applications" \
      -H "Content-Type: application/json" \
      -H "X-Correlation-ID: req-abc-123" \
      -d '{
            "legalName": "Jane Smith",
            "dateOfBirth": "1985-10-22",
            "nationality": "SE"
          }'
 ```
 
 **2. Fetch Application Status**
 ```bash
 curl -X GET "https://api.alborzbank.com/v1/applications/app_987654321" \
      -H "Authorization: Bearer <opaque_token>" \
      -H "X-Correlation-ID: req-abc-123"
 ```
