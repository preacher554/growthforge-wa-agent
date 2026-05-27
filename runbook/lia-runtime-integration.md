# Lia Runtime Integration v1

## Runtime repo

```txt
https://github.com/preacher554/whatsapp-agent-architect-runtime
/root/repos/whatsapp-agent-architect-runtime
```

## Runtime service

```txt
lia-runtime.service
```

Health:

```txt
http://127.0.0.1:3300/health
```

## Flow

```txt
WhatsApp customer message
↓
Evolution API instance: lia-growthforge
↓
Webhook: http://host.docker.internal:3300/webhook/evolution
↓
Lia runtime
↓
Supabase persistence
↓
Hermes OpenAI OAuth brain
↓
Evolution sendText reply
```

## Supabase

Project ref:

```txt
uxjcvxdmmkvwdnxqcgxb
```

Runtime uses Supabase transaction pooler via `DATABASE_URL` stored in:

```txt
/root/services/evolution-growthforge/supabase.env
```

## Evolution

Gateway stack:

```txt
/root/services/evolution-growthforge
```

Instance:

```txt
lia-growthforge
```

State after linking:

```txt
open
```

## Model routing

Lia runtime uses Hermes CLI with:

```txt
provider: openai-codex
model: gpt-5.4-mini
```

This follows Chief's decision: Lia uses OpenAI OAuth, not OpenRouter.

## MVP behavior

- Ignore messages sent by the linked WA account itself.
- Process one-to-one WhatsApp chats only.
- Save inbound/outbound messages to Supabase.
- Generate Indonesian reply as Lia.
- Detect early custom/price/integration/meeting/payment style handoff triggers.
- On handoff, set conversation to `waiting_human` and tell customer the team will handle it.

## Current limitations

- Admin private WA notification is not active until `admin_private_jid` is configured.
- Manual human resume is schema-ready but does not yet have an admin inbox UI.
- Direct Supabase URL may fail on IPv6 from this VPS; transaction pooler is valid and used.
