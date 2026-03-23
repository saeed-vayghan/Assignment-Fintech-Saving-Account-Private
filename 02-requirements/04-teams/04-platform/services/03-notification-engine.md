# Notification Engine

## What is it?
An asynchronous, event-driven service specializing in omni-channel customer communications. Instead of domain services (like Core Ledger) hardcoding email templates or managing SMTP servers, they simply publish standard Domain Events (like `DepositCreated`). The Notification Engine reacts to those events and handles the physical delivery of SMS, emails, or push notifications.

## Core Logic & Rules
1. **Event-Driven Choreography:** The entire engine runs in the background. It subscribes natively to the central AWS EventBridge bus, isolating communication failures from core banking processes.
2. **Strict Delivery Idempotency:** Users must absolutely never receive the same "Account Matured!" email twice. The engine utilizes DynamoDB to lock delivery and suppress any duplicate event payloads.
3. **Provider Agnostic:** It abstracts the downstream messaging providers (AWS SES, SNS, Twilio) away from the rest of the bank.

## Data Flow Visualization
```mermaid
flowchart TD
    %% Colors
    classDef ext fill:#f9e1e1,stroke:#d62828,stroke-width:2px,color:#000
    classDef internal fill:#e2ece9,stroke:#2a9d8f,stroke-width:2px,color:#000
    classDef engine fill:#e2ece9,stroke:#2a9d8f,stroke-width:2px,color:#000
    classDef db fill:#c0efd1,stroke:#264653,stroke-width:2px,color:#000
    classDef event fill:#fff3b0,stroke:#e9c46a,stroke-width:2px,color:#000

    Domain1["Team 2: Deposits"]:::internal -- "1a. DepositCreated" --> Bus(("🚌 Central EventBridge")):::event
    Domain2["Team 1: Onboarding"]:::internal -- "1b. CustomerVerified" --> Bus
    
    Bus -- "2. Routes Event" --> Logic["⚙️ Notification Engine<br/>(Event Handler)"]:::engine
    
    Logic -- "3. Acquires Delivery Lock" --> DB[("⚡ DynamoDB:<br/>Notification_Log")]:::db
    
    Logic -- "4. Fetches User Preferences" --> DB
    
    Logic -- "5. Dispatches Email/SMS" --> SES(("AWS SES / SNS")):::ext
    
    SES -- "6. Delivery Receipt Webhook" --> Logic
    
    Logic -- "7. Marks Status DELIVERED" --> DB
```
