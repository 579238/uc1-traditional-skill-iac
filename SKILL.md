---
name: uc1-traditional-case-lookup-iac
description: Create a traditional Lex-only voice bot for case/status lookup using AWS Connect, Lex, Lambda, and DynamoDB with 100% Infrastructure as Code (CloudFormation). No AI/Bedrock - pure deterministic IVR. Single stack deploys everything including Connect resources. Use when user wants to build a status check bot, claim lookup, application tracker, or case status system without AI components.
---

# UC1 Traditional: Case/Status Lookup Bot (Lex-Only) - 100% IaC

Deploy a voice-based case status lookup system using Amazon Connect + Lex + Lambda + DynamoDB. This is the **traditional approach** - no AI, no Knowledge Base, just deterministic IVR logic.

**100% Infrastructure as Code** - Single CloudFormation stack deploys everything. No post-deployment API calls needed.

## When to Use This Skill

- User wants a **status lookup** bot (claims, permits, applications, tickets)
- User wants **Lex-only** approach (no Bedrock/AI)
- User needs simple **ID + verification → status** flow
- User wants **100% Infrastructure as Code** deployment
- User wants **single stack, single delete** for easy management

## Architecture

```
Caller → Phone Number → Contact Flow → Lex Bot → Lambda → DynamoDB
              ↑              ↑            ↓
         (CFN created)  (CFN created)  4 Intents:
                                       - CheckStatus (ID + verification)
                                       - SMSOptIn (hardcoded response)
                                       - EndConversation
                                       - FallbackIntent
```

## What Gets Created (All via CloudFormation)

| Resource | CFN Resource Type |
|----------|-------------------|
| IAM Role (Lambda) | AWS::IAM::Role |
| DynamoDB Tables (2) | AWS::DynamoDB::Table |
| Lambda Function | AWS::Lambda::Function |
| Lambda Permission | AWS::Lambda::Permission |
| Lex V2 Bot + Alias | AWS::Lex::Bot, BotVersion, BotAlias |
| Connect Bot Association | AWS::Connect::IntegrationAssociation |
| Contact Flow | AWS::Connect::ContactFlow |
| Phone Number | AWS::Connect::PhoneNumber |
| Phone-Flow Association | Custom::PhoneNumberFlowAssociation |

## Prerequisites

- AWS Account with appropriate permissions
- Amazon Connect instance (already created)
- Connect instance ARN ready
- Lex service-linked role exists (find suffix in IAM → Roles → search "AWSServiceRoleForLexV2Bots")

---

## Step 1: Gather Use Case Details

Ask the user for:

| Question | Example Answer | Variable |
|----------|---------------|----------|
| What entity are you looking up? | "veteran", "applicant", "motorist" | `{{ENTITY_NAME}}` |
| What are you looking up for them? | "claim", "permit", "ticket" | `{{CASE_NAME}}` |
| What's the primary ID field? | "veteranId", "licensePlate" | `{{ENTITY_ID_FIELD}}` |
| What's the verification field? | "ssnLast4", "ticketNumber" | `{{VERIFICATION_FIELD}}` |
| AWS Region? | us-east-1 | `{{AWS_REGION}}` |
| Connect Instance ARN? | arn:aws:connect:us-east-1:123456789012:instance/abc-123 | `{{CONNECT_INSTANCE_ARN}}` |

Generate names:
- `{{USE_CASE_NAME}}`: e.g., "va-benefits", "parking-ticket"
- `{{STACK_NAME}}`: e.g., "va-benefits-bot", "parking-ticket-bot"
- `{{CASE_NAME_UPPER}}`: Uppercase first letter of case name (e.g., "Claim", "Card")

---

## Step 2: Generate CloudFormation Template

Copy `assets/cloudformation.yaml.template` and replace all placeholders.

**Cross-platform instructions:**
- **macOS/Linux**: Use `sed` to replace placeholders
- **Windows**: Use PowerShell string replacement:
  ```powershell
  $template = Get-Content "assets/cloudformation.yaml.template" -Raw
  $template = $template -replace '\{\{USE_CASE_NAME\}\}', 'actual-value'
  # ... replace all placeholders
  $template | Out-File "output.yaml" -Encoding utf8
  ```
- **Or**: Read the template, replace placeholders programmatically, then write to a new file

**Important**: The template is ~30KB. When creating the output file, ensure you provide the complete file content.

**Placeholders to replace:**
- `{{USE_CASE_NAME}}` - e.g., "va-benefits"
- `{{ENTITY_NAME}}` - e.g., "veteran"
- `{{CASE_NAME}}` - e.g., "claim"
- `{{CASE_NAME_UPPER}}` - e.g., "Claim" (uppercase first letter)
- `{{ENTITY_ID_FIELD}}` - e.g., "veteranId"
- `{{VERIFICATION_FIELD}}` - e.g., "ssnLast4"
- `{{CONNECT_INSTANCE_ARN}}` - the full Connect instance ARN

---

## Step 3: Deploy CloudFormation Stack

```bash
# Deploy the stack (creates EVERYTHING in one operation)
aws cloudformation create-stack \
  --stack-name {{STACK_NAME}} \
  --template-body file://{{USE_CASE_NAME}}-stack.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=UseCaseName,ParameterValue={{USE_CASE_NAME}} \
    ParameterKey=EntityName,ParameterValue={{ENTITY_NAME}} \
    ParameterKey=CaseName,ParameterValue={{CASE_NAME}} \
    ParameterKey=EntityIdField,ParameterValue={{ENTITY_ID_FIELD}} \
    ParameterKey=VerificationField,ParameterValue={{VERIFICATION_FIELD}} \
    ParameterKey=ConnectInstanceArn,ParameterValue={{CONNECT_INSTANCE_ARN}}

# Wait for stack completion (~5-8 minutes)
aws cloudformation wait stack-create-complete --stack-name {{STACK_NAME}}

# Get outputs including phone number
aws cloudformation describe-stacks \
  --stack-name {{STACK_NAME}} \
  --query 'Stacks[0].Outputs' \
  --output table
```

**Stack creates ALL resources (~5-8 min):**
- IAM Roles (Lambda + Lex + phone association)
- 2 DynamoDB tables (Entity + Cases with GSI)
- Lambda functions (fulfillment + phone association helper)
- Lambda permission for Lex
- Lex V2 Bot with 4 intents
- Lex Bot Version + Alias
- Connect Bot Association
- Contact Flow
- Phone Number (claimed and associated with flow)
- **Test data seeded automatically**

---

## Step 4: Get Phone Number and Test

```bash
# Get the phone number from stack outputs
PHONE_NUMBER=$(aws cloudformation describe-stacks --stack-name {{STACK_NAME}} \
  --query "Stacks[0].Outputs[?OutputKey=='PhoneNumber'].OutputValue" --output text)

echo "Bot ready! Call: $PHONE_NUMBER"
```

**Test data is already seeded!** Default test credentials:
- Entity ID: `12345`
- Verification: `1234`

---

## Step 5: Test

1. Call the phone number
2. Say "check my status"
3. Provide entity ID: **12345**
4. Provide verification: **1234**
5. Hear status response
6. Say "yes" to SMS opt-in
7. Hear confirmation
8. Say "goodbye"

---

## Cleanup / Rollback

**Delete everything with ONE command:**
```bash
# Delete the entire stack - removes ALL resources including Connect
aws cloudformation delete-stack --stack-name {{STACK_NAME}}
aws cloudformation wait stack-delete-complete --stack-name {{STACK_NAME}}
```

That's it! CloudFormation handles cleanup of all resources including phone number, contact flow, and bot association.

---

## Verification Checklist

- [ ] Stack deployed successfully (CREATE_COMPLETE) - ~5-8 min
- [ ] DynamoDB has test data
- [ ] Phone number output shows in stack outputs
- [ ] Test call works end-to-end

---

## Output Artifacts

After deployment, user receives:
1. **CloudFormation template** (`{{USE_CASE_NAME}}-stack.yaml`) - version control this
2. **Stack outputs** with all resource ARNs including phone number
3. **Working phone number** ready to call

---

## Troubleshooting

See `references/gotchas.md` for common issues.

**100% IaC-specific issues:**
- **Lex service role not found**: Get the correct suffix from IAM console
- **Stack rollback on Lex**: Usually NLU confidence threshold or voice settings issue
- **Phone number not claimed**: Check if toll-free numbers are available in region
- **Contact flow association fails**: Verify Connect instance ARN is correct

Key notes:
- Lex service-linked role must exist before stack deployment
- Stack takes 5-8 minutes due to Lex bot build + phone number claim
- Phone association uses a custom Lambda resource (automatic cleanup on delete)
