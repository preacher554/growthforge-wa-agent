# Handoff Runtime Design v1

## Default model

GrowthForge WA Agent uses **admin notification + same-number human reply**.

The customer continues chatting with the business WhatsApp number. When the AI cannot or should not answer, it pauses and notifies the tenant admin on the admin's private WhatsApp number.

The admin then replies to the customer using the same business WhatsApp/agent number, so the customer experience stays inside one official business identity.

## Flow

```txt
1. Customer sends message to business WhatsApp number.
2. Agent receives message through gateway.
3. Agent checks package scope, tenant knowledge, confidence, and escalation rules.
4. If no escalation is needed, AI replies normally.
5. If escalation is needed:
   a. AI sends a short waiting/handoff message to the customer.
   b. Runtime creates a handoff summary.
   c. Runtime sends notification to admin_private_whatsapp.
   d. Runtime sets conversation state to waiting_human.
   e. AI stops replying to that customer conversation.
6. Admin replies to customer from the same business WhatsApp/agent number.
7. Runtime marks state as human_active.
8. AI resumes only after manual release or configured timeout policy.
```

## Conversation states

```yaml
ai_active:
  description: AI can respond normally.

waiting_human:
  description: Handoff notification sent. AI should remain silent except approved waiting messages.

human_active:
  description: Human admin has taken over. AI must stay silent.

resolved:
  description: Human/admin closed the issue. AI can resume if released.
```

## Admin notification payload

```yaml
event_type: handoff_requested
tenant_id:
conversation_id:
customer_name:
customer_phone:
escalation_reason:
lead_status:
last_message:
summary:
recommended_next_action:
ai_state: waiting_human
reply_instruction: "Balas customer dari nomor WhatsApp bisnis/agent yang sama."
```

## Why this model

- Customer stays on the same WhatsApp number.
- Business identity remains consistent.
- AI does not argue or hallucinate when unsure.
- Admin receives context before replying.
- Basic/Pro stay simple without full CRM requirement.

## MVP requirement

The first runtime must support:

- per-tenant `admin_private_whatsapp`,
- per-conversation state,
- AI pause on handoff,
- admin notification message,
- manual or timeout-based resume policy.
