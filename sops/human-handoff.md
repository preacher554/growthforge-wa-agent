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

## Handoff message to customer

```txt
Baik Kak, untuk bagian itu aku bantu teruskan ke admin ya supaya jawabannya lebih tepat. Mohon tunggu sebentar.
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
