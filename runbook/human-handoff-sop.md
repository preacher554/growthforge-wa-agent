# Human Handoff SOP

## Goal

Ensure the AI stops and escalates when human authority, missing information, or custom judgment is required.

## Handoff triggers

Always handoff when customer asks for:

- final custom price,
- discount approval,
- refund/cancellation,
- complaint/escalation,
- legal/medical/financial decision,
- partnership/custom deal,
- unavailable/missing business info,
- angry customer,
- integration/custom workflow,
- payment issue.

## Basic package handoff

Basic should handoff early when conversation moves beyond FAQ or simple booking/order intake.

## Pro package handoff

Pro may qualify the lead first, then handoff with summary when:

- lead is hot,
- customer requests next step,
- custom pricing is needed,
- customer asks for human confirmation,
- AI confidence is low.

## Primary handoff model

The default GrowthForge handoff model is **admin notification + same-number human reply**:

```txt
customer chats business WhatsApp number
↓
AI detects handoff trigger
↓
AI tells customer it will forward to admin
↓
system sends handoff notification to the owner's/admin's private WhatsApp number
↓
AI pauses this conversation
↓
human admin replies to the customer using the same business WhatsApp/agent number
↓
AI stays paused until released or timeout policy resumes it
```

This means the customer should not be moved to a different number unless the tenant explicitly wants that. The business identity stays on the same WhatsApp number.

## Handoff message to customer

```txt
Baik Kak, untuk bagian itu aku bantu teruskan ke admin ya supaya jawabannya lebih tepat. Mohon tunggu sebentar.
```

## Admin notification message

```txt
[WA Agent Handoff]
Tenant: {{tenant_id}}
Customer: {{customer_name}} / {{customer_phone}}
Reason: {{escalation_reason}}
Lead status: {{lead_status}}
Last message: {{last_message}}

Recommended next action:
{{recommended_next_action}}

AI sudah pause untuk chat ini. Silakan balas customer dari nomor WhatsApp bisnis/agent yang sama.
```

## Conversation pause states

```txt
ai_active      = AI may respond normally.
waiting_human  = escalation sent; AI must not answer customer except approved waiting message.
human_active   = admin has taken over; AI stays silent.
resolved       = issue closed; AI may resume if explicitly released or timeout policy allows.
```

## Internal summary format

```yaml
tenant_id:
customer_name:
customer_phone:
intent:
need:
lead_status:
context:
last_message:
escalation_reason:
recommended_next_action:
```
