# Lia Human Takeover + 1-Hour Resume Policy

Session-derived, user-approved behavior for GrowthForge Lia when Chief/admin handles a customer manually from the same WhatsApp Business number used by Lia.

## Scenario

A customer reaches a handoff condition (custom need, budget/deal, demo, integration, or human follow-up). Lia sends the handoff message and pauses. Chief/admin then opens the same WhatsApp Business app/number and replies manually to the customer. After the human finishes, Lia should be able to resume later without interrupting the human conversation.

## Desired behavior

```txt
Lia handoff reply
→ state = waiting_human
→ human/admin sends message from same WA Business number
→ Evolution event has key.fromMe = true
→ runtime records human takeover
→ state = human_active
→ 1-hour timer starts/resets from latest human message
→ customer messages before timer expiry are stored but not answered by Lia
→ customer message after timer expiry makes Lia eligible to answer
→ Lia replies using recent context, not a fresh intro
```

## Important nuance

The 1-hour timer should not cause Lia to proactively message the customer. It only makes the conversation eligible for AI reply on the next customer inbound message.

Correct:

```txt
10:56 human/admin last reply
11:56 conversation eligible for AI
12:10 customer sends new message
12:10 Lia replies naturally with context
```

Avoid:

```txt
11:56 timer expires
11:56 Lia sends unsolicited follow-up automatically
```

## Runtime implementation notes

- Evolution must emit sent-message events for same-number admin replies. Current docker-compose pattern enables `WEBHOOK_EVENTS_SEND_MESSAGE=true` plus `WEBHOOK_EVENTS_MESSAGES_UPSERT=true`.
- Do not globally drop `fromMe:true` before conversation lookup. `fromMe:true` should bypass model generation, but it is still operationally meaningful for state transitions.
- For paused conversations:
  - `waiting_human + fromMe:true` → `human_active`
  - update/reset `last_human_message_at` or equivalent metadata
  - persist the human message as outbound/human if the schema supports it
- For inbound customer messages while `human_active`:
  - if `now < last_human_message_at + 1 hour`: store and return paused
  - if `now >= last_human_message_at + 1 hour`: set `ai_active`/resume and generate reply from history
- New `fromMe:true` events always reset the timer.

## Product UX rationale

This keeps human takeover safe: Lia does not cut into an active manual sales/deal conversation, but she can continue covering the customer later if the human window has gone cold. For Chief's workflow, this is the preferred default over immediate auto-resume.
