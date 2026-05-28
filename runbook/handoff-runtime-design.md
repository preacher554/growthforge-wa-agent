# Handoff Runtime Design v2

> Upgraded from v1 after Lia production validation. Adds `fromMe` classifier, outbox echo handling, 1-hour human window with context-preserving resume.

## Default model

GrowthForge WA Agent uses **admin notification + same-number human reply**.

The customer continues chatting with the business WhatsApp number. When the AI cannot or should not answer, it pauses and notifies the tenant admin on the admin's private WhatsApp number.

The admin then replies to the customer using the same business WhatsApp/agent number, so the customer experience stays inside one official business identity.

## Flow (v2)

```txt
1. Customer sends message to business WhatsApp number.
2. Agent receives message through Evolution webhook.
3. Agent dedupes, checks package scope, tenant knowledge, confidence, and escalation rules.
4. If no escalation is needed, AI replies normally via outbox.
5. If escalation is needed:
   a. AI generates short handoff reply → enqueue outbox → send to customer.
   b. Runtime creates handoff summary.
   c. Runtime sends notification to admin_private_whatsapp.
   d. Runtime transitions state: waiting_human, owner: human.
   e. AI stops auto-replying to that conversation.
6. Admin replies to customer from the same business/number.
   a. Evolution webhook arrives with key.fromMe = true.
   b. Runtime checks: is this an outbox echo (Lia's own outbound) or human admin message?
   c. If human admin message: persist, transition state → human_active, record last_human_activity_at.
7. During 1-hour human window:
   a. Customer inbounds within 1 hour of last human message → stored, no AI reply.
   b. New fromMe:true events reset the 1-hour timer.
   c. Timer expiry does NOT cause Lia to send a proactive message.
8. Resume on next customer inbound after 1-hour window:
   a. Runtime transitions state → ai_active, owner: ai.
   b. Resume note injected into model prompt (continue from context, not restart).
   c. Human outbound messages labeled as "Admin [TenantName]" in history, not as AI replies.

## Same-number fromMe classifier

Critical mechanism: admin and AI share one WhatsApp number. Evolution emits sent-message events with `key.fromMe: true`.

v1 handled this with a simple drop. v2 requires conversational awareness:

```txt
fromMe: true received
  ↓
Match against outbox_messages (pending/sent)?
  ├─ Yes → This is Lia's own outbound echo → mark outbox sent, no action
  └─ No → This is human admin message
       ├─ Persist as outbound (source: human_admin)
       ├─ Transition waiting_human → human_active
       ├─ Update last_human_activity_at
       ├─ Emit ownership.changed → human
       └─ Return without AI generation
```

Pitfall: do not globally drop `fromMe: true` before conversation lookup. The event is operationally meaningful for state transitions.

## Ownership model

v2 introduces an **ownership** dimension separate from lifecycle state:

| Owner | Meaning |
|---|---|
| `ai` | AI is allowed to generate replies |
| `human` | Human admin has taken over; AI must stay silent |
| `system` | System-level actions (init, cleanup, scheduled resume) |

Ownership enforces hard boundaries: when `owner: human`, no model call is made regardless of inbound volume.

## Conversation states (v2, expanded from v1)

```yaml
ai_active:
  description: AI can respond normally. Default state.

waiting_human:
  description: Handoff notification sent. AI must not answer customer except waiting messages.

human_active:
  description: Human admin has taken over via fromMe:true. AI stays silent. 1-hour window starts.

resolved:
  description: Human/admin closed the issue. AI can resume if released or timeout policy allows.

idle:
  description: Conversation exists but no active processing.

user_buffering:
  description: Accumulating short successive messages before AI planning.

ai_planning:
  description: Model call in progress for reply generation.

ai_sending:
  description: Reply generated, sending via outbox worker.

failed_recoverable:
  description: Transient failure (model timeout, network). Will retry.

suspended:
  description: Admin or system policy suspended this conversation.
```

## 1-hour human window + resume policy

Fixed behavior validated in Lia production:

1. Lia escalates and sets conversation to `waiting_human`.
2. Admin replies manually from the same WhatsApp Business number.
3. Runtime detects `fromMe: true`, transitions to `human_active`, records `last_human_activity_at`.
4. During the 1-hour human window, customer messages are stored but Lia stays silent.
5. Timer expiry does NOT trigger an outbound message. It only marks the conversation as "eligible for resume on next inbound".
6. When the customer sends a new message after the 1-hour window:
   - Runtime transitions `ai_active`, owner: `ai`
   - Resume context injected into prompt:
     > Percakapan ini baru di-resume otomatis setelah admin/human GrowthForge mengambil alih.
     > Balas sebagai WA Agent yang aktif kembali. Jangan ulang dari awal; lanjutkan natural dari konteks chat.
7. Any new `fromMe: true` (admin message) resets the 1-hour timer.
8. Resume should feel natural — not a fresh greeting, but a contextual continuation.

Evolution prerequisites:
- `WEBHOOK_EVENTS_SEND_MESSAGE=true` (to receive same-number human replies)
- `WEBHOOK_EVENTS_MESSAGES_UPSERT=true`
- Same WhatsApp instance configured for both AI outbound and human admin replies

## Handoff keyword detection

Handoff keywords trigger `waiting_human` transition. v2 specificity rules:

- Use **word boundary matching** to avoid false positives.
- Keyword `custom` must NOT match inside `customer`, `customize`, or similar.
- Compile keyword list per tenant from `handoff-rules.md` tenant config.

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

## Resume prompt engineering

When Lia resumes after human takeover, the history fed to the model must:

1. **Distinguish human admin messages from AI messages**: Label `fromMe: true` outbound as `Admin [TenantName]`, not as the AI agent. This prevents the model from treating the admin's negotiation/closing messages as its own prior replies.
2. **Inject resume context**: Operational note (not customer-visible) explaining the resume situation.
3. **Preserve full conversation window**: Include the human admin's messages so the model understands what was said during the AI pause.
4. **Avoid restart loops**: If history contains 3+ messages from the human window, acknowledge them naturally instead of re-asking name/business.

Pitfall: if the model sees its own earlier greeting after a long human window, and the resume prompt is missing, it may re-introduce itself. The resume note prevents this.

## Why v2 over v1

| Aspect | v1 | v2 |
|---|---|---|
| fromMe handling | Global drop | Conversational classifier with outbox matching |
| Timer behavior | Manual release or vague timeout | 1-hour fixed window, no proactive send |
| Resume quality | Restart risk | Context-preserving with prompt engineering |
| Ownership | Implicit in state | Explicit Owner dimension |
| State coverage | 4 states | 10 lifecycle states |
| Production validated | No | Yes, in Lia (GrowthForge) |

## MVP requirement (v2)

Every WA Agent runtime must support:

- per-tenant `admin_private_whatsapp`
- per-conversation lifecycle state + ownership
- `fromMe: true` classification (outbox echo vs human admin)
- AI pause on handoff with ownership lock
- admin notification message
- 1-hour human window with no-proactive-send semantics
- context-preserving resume on next customer inbound
- word-boundary handoff keyword detection
