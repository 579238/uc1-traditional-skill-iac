# Architecture: Traditional Case Lookup Bot (Lex-Only)

## Overview

Pure deterministic IVR using Amazon Connect, Lex, Lambda, and DynamoDB. No AI components.

```
┌─────────────────────────────────────────────────────────────────┐
│                         CALLER                                  │
│                     (Phone Call)                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AMAZON CONNECT                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Contact Flow                                │   │
│  │  1. Play welcome message                                 │   │
│  │  2. Get customer input → Lex Bot                        │   │
│  │  3. Loop until EndConversation                          │   │
│  │  4. Disconnect                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      AMAZON LEX V2                              │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐            │
│  │ CheckStatus  │ │  SMSOptIn    │ │EndConversation│           │
│  │              │ │              │ │              │            │
│  │ Slots:       │ │ No slots     │ │ No slots     │            │
│  │ - entityId   │ │              │ │              │            │
│  │ - verifyCode │ │ Utterances:  │ │ Utterances:  │            │
│  │              │ │ "yes"        │ │ "goodbye"    │            │
│  │ Utterances:  │ │ "sure"       │ │ "bye"        │            │
│  │ "check status│ │ "text me"    │ │ "I'm done"   │            │
│  └──────────────┘ └──────────────┘ └──────────────┘            │
│                          │                                      │
│                   FallbackIntent                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AWS LAMBDA                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  handler(event, context)                                 │   │
│  │                                                          │   │
│  │  Routes by intent:                                       │   │
│  │  - CheckStatus → verify + lookup                        │   │
│  │  - SMSOptIn → hardcoded response                        │   │
│  │  - EndConversation → goodbye                            │   │
│  │  - FallbackIntent → help message                        │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AMAZON DYNAMODB                              │
│  ┌────────────────────────┐  ┌────────────────────────────┐    │
│  │     EntityTable        │  │       CasesTable           │    │
│  │                        │  │                            │    │
│  │  PK: entityId          │  │  PK: caseId                │    │
│  │                        │  │  GSI: entityId-index       │    │
│  │  Attributes:           │  │                            │    │
│  │  - name                │  │  Attributes:               │    │
│  │  - verificationField   │  │  - entityId                │    │
│  │  - phone               │  │  - status                  │    │
│  │                        │  │  - submittedDate           │    │
│  └────────────────────────┘  └────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

## Conversation Flow

```
1. Caller: "Check my claim status"
   → Lex: CheckStatus intent detected
   
2. Lex: "Please provide your ID number"
   Caller: "12345"
   → Slot: entityId = "12345"
   
3. Lex: "Please provide last 4 of your SSN"
   Caller: "5678"
   → Slot: verificationCode = "5678"
   → Fulfillment triggered
   
4. Lambda: verify_entity() + get_cases()
   → "Your claim is in-review. Would you like SMS notifications?"
   
5. Caller: "Yes"
   → Lex: SMSOptIn intent
   → Lambda: hardcoded response
   → "You're set for notifications. Anything else?"
   
6. Caller: "No, goodbye"
   → Lex: EndConversation intent
   → "Thank you for calling. Goodbye!"
   → Disconnect
```

## Key Differences from Hybrid

| Aspect | Traditional | Hybrid |
|--------|-------------|--------|
| FAQ Handling | FallbackIntent only | Bedrock KB |
| AI Components | None | Claude for KB queries |
| Intents | 4 (Check, SMS, End, Fallback) | 5 (+ AskQuestion) |
| Lambda Deps | boto3 only | boto3 + bedrock-agent-runtime |
| IAM Permissions | DynamoDB only | DynamoDB + Bedrock |
