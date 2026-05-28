# Human Handoff SOP (v2)

## Goal

Ensure the AI stops and escalates when human authority, missing information, or custom judgment is required — and resumes intelligently after human takeover.

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

## Handoff keyword detection

Handoff triggers can also be activated by configured keywords (from tenant `handoff-rules.md`).

v2 rules:
- Use **word boundary matching** to prevent false positives.
- Keyword `custom` must NOT match inside `customer`, `customize`, or similar.
- Test regex: `\bcustom\b` matches standalone "custom" but not "customer".

## Basic package handoff

Basic should handoff early when conversation moves beyond FAQ or simple booking/order intake.

## Pro package handoff

Pro may qualify the lead first, then handoff with summary when:

- lead is hot,
- customer requests next step,
- custom pricing is needed,
- customer asks for human confirmation,
- AI confidence is low.

## Primary handoff model (v2)

The default GrowthForge handoff model is **admin notification + same-number human reply**:

```txt
customer chats business WhatsApp number
↓
AI detects handoff trigger
↓
AI tells customer it will forward to admin
↓
AI sends handoff reply via outbox → state: waiting_human → owner: human
↓
system sends handoff notification to the owner's/admin's private WhatsApp number
↓
human admin replies to the customer using the same business/number
↓
Evolution webhook: fromMe: true → runtime classifies as human admin message
↓
Runtime persists human message, transitions state → human_active
↓
last_human_activity_at = now()
↓
customer inbounds within 1 hour → stored, no AI reply
↓
customer inbound after 1 hour → ai_active, context-preserving resume
↓
any new fromMe:true resets the 1-hour timer
```

The customer stays on the same WhatsApp number. Business identity remains consistent.

## Handoff message to customer

```txt
Baik Kak, untuk bagian itu aku bantu teruskan ke admin ya supaya jawabannya lebih tepat. Mohon tunggu sebentar.
```

Include business hours if relevant (e.g., admin available 09:00-17:00 WIB). Don't ask when the user is free.

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

## Conversation states + ownership (v2)

Two orthogonal dimensions:

**Lifecycle states:**
```txt
ai_active       = AI may respond normally
waiting_human   = escalation sent; AI must not answer except waiting message
human_active    = admin replying; AI stays silent, 1-hour window active
resolved        = issue closed; AI may resume on next inbound
```

**Ownership:**
```txt
owner: ai       = AI is allowed to act
owner: human    = Human has taken over; AI hard-locked out
owner: system   = System operations only
```

When `owner: human`, NO model call is made regardless of inbound volume. This is a hard boundary.

## 1-hour human window — critical rules

1. **No proactive send on timer expiry.** The 1-hour window only makes the conversation eligible for AI reply on the next customer inbound. It must NEVER cause the AI to send an unsolicited message.

2. **Timer resets on human activity.** Any new `fromMe: true` (admin reply) resets the 1-hour clock.

3. **Customer messages during the window are stored.** They are saved to the DB for context continuity but do not trigger AI replies.

4. **Resume is context-preserving.** The next customer inbound after the window:
   - Transitions to `ai_active`, owner: `ai`
   - Injects a resume note into the model prompt (see below)
   - Does NOT restart the conversation from a greeting

## Resume prompt engineering (v2)

When building model context after human takeover:

1. Label human admin outbound messages as `Admin [TenantName]`, NOT as the AI agent.
2. Inject a system-level operational note:
   ```
   Percakapan ini baru di-resume otomatis setelah admin/human GrowthForge mengambil alih.
   Balas sebagai WA Agent yang aktif kembali. Jangan ulang dari awal; lanjutkan natural dari konteks chat.
   Jika cocok, awali singkat dengan 'Aku [nama agent] bantu lanjut ya Kak.'
   ```
3. Include the full conversation window including human admin messages.
4. If history is non-empty on resume, never ask name/business again.

## Outbox echo handling

Lia's own outbound messages arrive as `fromMe: true` when Evolution delivers sent-message events. v2 distinguishes:

```txt
fromMe: true
  ↓
Matches a pending outbox row?
  ├─ Yes → Mark outbox as sent (echo) → no state change
  └─ No → Human admin message → persist + state transition
```

Without this, every Lia reply would be misclassified as human takeover.

## QA checklist

- [ ] Handoff keyword `custom` does NOT match `customer`
- [ ] Lia stops replying after handoff (state: waiting_human)
- [ ] Admin notification sent to private WA
- [ ] fromMe: true correctly classified (echo vs human)
- [ ] 1-hour window: customer messages stored, no AI reply
- [ ] 1-hour timer expiry: NO proactive send
- [ ] Resume after timer: contextual, not a restart
- [ ] New fromMe: true resets timer
- [ ] Human outbound messages labeled as "Admin" in model history
- [ ] Outbox echo does not trigger human_active transition
