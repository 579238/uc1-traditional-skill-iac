# Gotchas: Traditional Case Lookup Bot (Lex-Only)

Common issues for Lex-only deployments. No KB-specific issues here.

---

## DynamoDB

### 1. GSI required for case lookup by entity
The Cases table needs a Global Secondary Index on the entity ID field:
```bash
--global-secondary-indexes '[{
  "IndexName":"{entityId}-index",
  "KeySchema":[{"AttributeName":"{entityId}","KeyType":"HASH"}],
  "Projection":{"ProjectionType":"ALL"}
}]'
```

### 2. Decimal types in DynamoDB
Numbers return as `Decimal`. Always use `decimal_to_native()` helper.

### 3. Use numeric IDs for IVR
Keep entity IDs and case IDs as 4-5 digit numbers. Callers can't easily speak alphanumeric IDs over phone.

---

## Lambda

### 4. Voice input comes as words
"one two three four" → needs parsing. Use `parse_digits()` to extract numbers.

### 5. Verification field comparison
Always normalize both stored and input values before comparing.

### 6. Use Python 3.14
Latest runtime - specify `--runtime python3.14` when creating Lambda.

### 7. Keep conversation open after status
Return message asking "Is there anything else?" to allow SMS opt-in flow.

---

## Lex Bot

### 8. Only 2 slots for CheckStatus
- entityId (priority 1)
- verificationCode (priority 2)

### 9. Slot chaining configuration
```json
{
  "slotName": "entityId",
  "captureNextStep": {
    "dialogAction": {"type": "ElicitSlot", "slotToElicit": "verificationCode"}
  }
}
```

### 10. SMSOptIn has no slots
Just sample utterances like "yes", "sure", "send me a text". Lambda returns hardcoded response.

### 11. NO EndConversation in postFulfillmentStatusSpecification
This drops the call immediately. Just use `{"enabled": true}`.

### 12. Lambda permission after alias update (CRITICAL)
Every time you rebuild bot and update alias version, re-add Lambda permission:
```bash
aws lambda add-permission \
  --function-name {fn} \
  --statement-id "lex-v{version}-$(date +%s)" \
  --action lambda:InvokeFunction \
  --principal lexv2.amazonaws.com \
  --source-arn {alias-arn}
```

---

## Connect

### 13. Phone claiming is 2-step
1. Claim number to instance
2. Associate number with contact flow (separate API call)

### 14. Bot association uses associate-bot
```bash
aws connect associate-bot \
  --instance-id {instance} \
  --lex-v2-bot "AliasArn={alias-arn}"
```

---

## Response Formatting

### 15. Personalize with first name
```python
first_name = entity.get('name', 'there').split()[0]
response = f"Okay {first_name}, your claim is..."
```

### 16. Format dates for speech
Convert "2024-06-15" to "June 15":
```python
date_obj = datetime.strptime(submitted, '%Y-%m-%d')
spoken_date = date_obj.strftime('%B %d').replace(' 0', ' ')
```

### 17. Handle multiple cases
Summarize first 3, don't list all:
```python
for i, case in enumerate(cases[:3], 1):
    ordinal = ['first', 'second', 'third'][i-1]
    response += f"Your {ordinal} is {status}. "
```

---

## Testing Checklist

- [ ] Lambda permission exists for current alias
- [ ] DynamoDB has test data with numeric IDs
- [ ] Test ID + verification in Lex console
- [ ] Test with non-existent ID → proper error
- [ ] Test with wrong verification → proper error
- [ ] Test SMS opt-in → hardcoded confirmation
- [ ] Test phone call - greeting plays
- [ ] Test phone call - full lookup flow
- [ ] Test phone call - SMS opt-in
- [ ] Verify "Is there anything else?" prompts
- [ ] Verify EndConversation works
