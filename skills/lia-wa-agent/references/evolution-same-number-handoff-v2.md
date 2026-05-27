# Evolution same-number handoff

This reference is mandatory when a runtime uses Evolution API or another Baileys-based WhatsApp bridge and wants the human admin to reply from the same business WhatsApp number.

## Core bug to catch

Generic chatbot tutorials often say: ignore messages sent by the linked account itself.

For WhatsApp handoff, this is wrong. Evolution emits `fromMe=true` for:

- AI messages sent by the runtime through Evolution.
- System messages sent by the runtime through Evolution.
- Human admin messages typed manually from the linked business WhatsApp number.

If the runtime ignores every `fromMe=true` event, it cannot detect human admin takeover.

## Required outbox dependency

Create an outbox ledger before sending through Evolution.

```sql
CREATE TABLE outbox_messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id TEXT NOT NULL,
  conversation_id TEXT NOT NULL,
  content TEXT NOT NULL,
  content_hash TEXT NOT NULL,
  source TEXT NOT NULL DEFAULT 'ai', -- ai | system
  status TEXT NOT NULL DEFAULT 'pending',
  provider_message_id TEXT,
  attempt_count INT NOT NULL DEFAULT 0,
  next_retry_at TIMESTAMPTZ,
  sent_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Classification algorithm

```text
on webhook.message:
  if fromMe == false:
    classify source = customer
    persist inbound
    enqueue conversation processing
    return 200

  if fromMe == true:
    match = find outbox row by provider_message_id
         OR by runtime request id
         OR by content_hash plus short time window

    if match exists:
      classify source = match.source       # ai or system
      persist outbound confirmation
      update outbox status if needed
      do not trigger AI planning
      return 200

    else:
      classify source = human_admin
      persist human outbound message
      set conversation.owner = 'human'
      set conversation.state = 'human_active'
      set last_human_activity_at = now()
      emit human.outbound_detected
      suppress all AI replies until release
      return 200
```

## Resume flow

```text
human_active
  -> admin command RESUME <conversation_id>
  -> resume_pending
  -> build continuity summary from human session messages
  -> inject summary into next AI context
  -> idle with owner='ai'
```

Timeout-based release is allowed, but manual release should exist even before an admin UI. Use a private admin WhatsApp command as the MVP.

## Tests to require

- AI outbox send produces `fromMe=true`; runtime classifies it as AI and does not pause itself.
- Human admin manually replies from business number; runtime classifies it as human and sets owner to `human`.
- Customer sends another message during human ownership; AI stays silent.
- Admin sends `RESUME <conversation_id>` from private number; runtime writes continuity summary before AI resumes.
- Duplicate `fromMe=true` webhook does not create duplicate messages or state transitions.
