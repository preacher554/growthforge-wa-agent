---
name: lia-wa-agent
description: Lia — GrowthForge WhatsApp AI receptionist + sales agent. Handles multi-tenant WA conversations via Evolution API bridge, Supabase persistence, configurable Hermes model routing, and human handoff. Human-like sales receptionist flow with escalation to human team.
---

# Lia WA Agent — GrowthForge WhatsApp AI Receptionist

Lia is GrowthForge's front-line WhatsApp AI agent. She handles inbound customer conversations autonomously and escalates to human team when needed.

## Current Runtime

```txt
WhatsApp → Evolution API (instance: lia-growthforge) → Webhook POST /webhook/evolution
→ FastAPI Lia Runtime (port 3300) → Supabase persistence → Hermes CLI model reply → Evolution sendText
```

## Repos & Paths

```txt
Runtime:      /root/repos/whatsapp-agent-architect-runtime
Persona:      /root/repos/WA_Agent_Persona_Lia  (also on GitHub: preacher554/WA_Agent_Persona_Lia)
Master Skill: /root/repos/whatsapp-agent-architect
Evolution:    /root/services/evolution-growthforge
Supabase ref: uxjcvxdmmkvwdnxqcgxb
PM2 service:  growthforge-mission-monitor (dashboard port 3200)
Runtime Svc:  lia-runtime.service (systemd, port 3300)
```

## Persona Repo File Structure

The `WA_Agent_Persona_Lia` repo (local + GitHub) contains these tenant-specific files:

```txt
README.md            → repo description
business-profile.md  → Lia's business/product profile
faq.md               → FAQ for customer conversations
handoff-rules.md     → escalation/handoff rules
tenant.yaml          → tenant configuration (tenant_id, model, etc.)
```

When pushing persona changes to GitHub, use the GrowthForge-safe push workflow (see `github-auth` skill's `references/growthforge-env-notes.md`). Never `echo` the GitHub token in terminal — use `execute_code` for all token-bearing operations.

## Repo Architecture (v2)

The WA Agent system uses a 3-layer repo model. See `references/wa-agent-repo-architecture.md` for the full pattern, spawn workflow, and multi-platform setup.

Quick reference:
- `whatsapp-agent-architect` → master skill (shared, no tenant data)
- `WA_Agent_Persona_Lia` → per-agent persona repo (Lia-specific config)
- `whatsapp-agent-architect-runtime` → runtime code (webhook, Evolution bridge, dashboard)

To spawn a new customer agent: create `WA_Agent_Persona_<Name>` repo from Lia template, fill in client-specific config, connect via `tenant.yaml`.

## Model Config

Model/provider are set in `app/config.py` → `hermes_model_provider` + `hermes_model`. They can be overridden via env `HERMES_MODEL_PROVIDER` / `HERMES_MODEL` if the service environment sets them.

Current runtime after the May 2026 repair:
```txt
Provider: openai-codex
Model:    gpt-5.2
```

Before changing Lia's model, verify the target route with a direct Hermes CLI probe from the runtime repo:
```bash
cd /root/repos/whatsapp-agent-architect-runtime
hermes chat -Q --provider <provider> -m <model> -q 'Reply with exactly LIA_MODEL_OK and nothing else.'
```

Then edit `app/config.py`, restart Lia, and verify `/health` plus DB/outbound behavior.

**Routing guardrail:** Yuya/main operator should remain OAuth-first. Lia can be routed explicitly per business need, but do not assume OpenRouter is the only valid runtime path; verify the configured provider/model in `app/config.py` and live service env before diagnosing model failures.

**OpenRouter key**: Stored in `/root/.hermes/.env` as `OPENROUTER_API_KEY` when Lia uses OpenRouter. Loaded by lia-runtime.service via `EnvironmentFile=/root/.hermes/.env`. After key rotation: `systemctl restart lia-runtime.service`.

## Supabase Schema

```txt
tenants         → WA agent instances (growthforge_lia row)
conversations   → per-customer threads (tenant_id FK)
messages        → inbound/outbound/system messages (conversation_id FK)
handoff_events  → escalation records
lead_summaries  → lead qualification summaries
```

Customer ID = `remote_jid` (WhatsApp JID format: `628xxx@s.whatsapp.net`).

## Conversation Flow (USER-APPROVED)

See `references/lia-conversation-rules.md` for the canonical Lia system prompt and flow rules.

Key behaviors:
1. **Greeting is time-aware**: Pagi (05-11), Siang (11-15), Sore (15-18), Malam (18-05) — all WIB
2. **Introduce products BEFORE asking user needs**: WA Agent Basic → WA Agent Pro → InstaGrow
3. **Address as "Kak [nama]"**: Never bare name alone. "Kak" if name unknown.
4. **Escalation**: Immediately confirm follow-up + business hours (09:00-17:00 WIB). Don't ask when user is free.
5. **No meta-commentary**: Never output reasoning/instructions to customer
6. **Brevity**: Max 2-4 sentences, 1-2 questions per reply
7. **Package questions**: After name+business collected → ask main problem → volume → goals

## Conversation States

```txt
ai_active       = Lia auto-replying
waiting_human   = Paused, waiting for human takeover
human_active    = Human handling from WhatsApp
resolved        = Done
```

## Human Takeover + 1-Hour Resume Policy

User-approved operating model for same-number WhatsApp Business handoff:

1. Lia escalates and sets the customer conversation to `waiting_human`.
2. Chief/admin replies manually from the same WhatsApp Business number. Evolution should emit this as a message event with `key.fromMe = true` when `WEBHOOK_EVENTS_SEND_MESSAGE=true` / relevant sent-message webhook is enabled.
3. Runtime must not ignore `fromMe:true` completely. For paused conversations, use it as a human-takeover signal:
   - persist/record the outbound human message if appropriate,
   - change `waiting_human -> human_active`,
   - set/reset a 1-hour resume timer from the latest human message timestamp.
4. During the 1-hour human window, customer messages are stored but Lia stays silent.
5. After 1 hour from the last human/admin message, Lia becomes eligible to resume, but should **not proactively send a new message just because the timer expired**.
6. Lia resumes only when the customer sends a new inbound message after the 1-hour window, then answers using the full recent context so she continues naturally instead of restarting the sales flow.
7. Any new human/admin message resets the 1-hour timer.

Pitfall: if `IncomingMessage.should_process` rejects all `fromMe:true` before conversation lookup, Lia cannot know human takeover happened. Handle sent/fromMe events before the normal inbound-only auto-reply path.

Pitfall: if Evolution does not deliver same-number WhatsApp Business human replies as `fromMe:true`, paused conversations can remain in `waiting_human`. Runtime should allow a conservative fallback: when `waiting_human` has exceeded the 1-hour window since the latest handoff, the next customer inbound can resume Lia without proactive sending.

Pitfall: handoff keyword matching must use word boundaries. The keyword `custom` must not match inside `customer`; otherwise normal phrases like "1-2 customer" falsely trigger human handoff.

## Common Operations

### Disable all WA agents safely
Use all three layers when Chief asks to pause/nonaktifkan WA agents:
1. Evolution webhook off: `POST /webhook/set/{instance}` with `{"webhook":{"enabled":false,"events":[]...}}`
2. Stop/disable runtime: `systemctl stop lia-runtime.service && systemctl disable lia-runtime.service`
3. DB tenant flag off: `update tenants set ai_enabled=false`

Runtime also has a kill-switch config: `WA_AGENTS_ENABLED` / `wa_agents_enabled` (default false). When false, `/webhook/evolution` returns `{"ignored":"wa_agents_disabled"}` before inserting conversations/messages or sending replies.

### Clear all customer data (fresh test)
```bash
supabase.env → psql:
BEGIN;
DELETE FROM messages WHERE conversation_id IN (SELECT id FROM conversations);
DELETE FROM handoff_events WHERE conversation_id IN (SELECT id FROM conversations);
DELETE FROM lead_summaries WHERE conversation_id IN (SELECT id FROM conversations);
DELETE FROM conversations;
COMMIT;
```

### Check model currently in use
```bash
grep hermes_model /root/repos/whatsapp-agent-architect-runtime/app/config.py
```

### Change model
1. Edit `app/config.py` → `hermes_model`
2. Edit `src/app/api/wa-agents/route.ts` if hardcoded
3. `systemctl restart lia-runtime.service`
4. `npm run build` + `pm2 restart growthforge-mission-monitor --update-env` (dashboard)
5. `git add -A && git commit && git push` (both repos)

### Send test webhook
```bash
curl -X POST http://127.0.0.1:3300/webhook/evolution \
  -H 'Content-Type: application/json' \
  -d '{"event":"messages.upsert","instance":"lia-growthforge","data":{"key":{"remoteJid":"628xxx@s.whatsapp.net","fromMe":false,"id":"test"},"message":{"conversation":"Halo"},"messageTimestamp":1716576000,"pushName":"Test"}}'
```

### Check logs
```bash
journalctl -u lia-runtime.service -n 50 --no-pager
```

## Bundled references

- `references/lia-conversation-rules.md` — Canonical Lia system prompt and flow rules
- `references/lia-runtime-debugging.md` — Silent-reply diagnosis, dedupe ordering, model-switch verification
- `references/human-takeover-resume-policy.md` — Same-number handoff + 1-hour resume policy
- `references/lia-human-resume-implementation.md` — Runtime implementation details for human resume
- `references/wa-agent-repo-architecture.md` — 3-layer repo model (master skill / persona / runtime)
- `references/wa-agent-master-skill-content-gap-map.md` — Content gap analysis from external templates
- `references/state-machine-and-events.md` — 12-state formal state machine with audited transitions
- `references/evolution-same-number-handoff-v2.md` — Outbox-based `fromMe` classifier with SQL schema
- `references/runtime-capability-checklist.md` — Full audit checklist for WA AI runtimes
- `references/production-risks.md` — Risk taxonomy (critical/high/medium)
- `references/implementation-roadmap.md` — Phased rollout plan (Phase 0–6)
- `references/runtime-architecture.md` — Module & folder structure, queue topology, Kafka scale-up path
- `references/database-schema.md` — Full Supabase/Postgres schema: tenants, conversations, messages, outbox, buffers, signals, memories, dead letters, Redis keys, RLS
- `references/message-buffer-design.md` — Fragment buffering algorithm, merge rules, BullMQ/Redis ZSET patterns
- `references/humanization-and-pacing.md` — Response delay formula, typing indicator, long message chunking, burst protection
- `references/emotional-signals.md` — Emotional/urgency/lead temp/escalation risk signal schema, extraction flow, rule-based detection
- `references/continuity-and-memory.md` — Continuity summarizer, 4-layer memory system (conversation/customer/summary/business), context assembly order
- `references/circuit-breaker-and-resilience.md` — LLM exponential backoff, auto-escalation on exhaustion, stale lock recovery, Redis SETNX dedupe
- `templates/runtime-redesign-report.md` — Redesign audit report template
- `runbook/package-scope-playbook.md` — SOP lengkap: Basic/Pro/Custom scope, flow, QA checklist, intake form
- `runbook/writing-playbook.md` — **WhatsApp Writing Playbook**: struktur bubble, emoji rules, formatting, template per situasi
- `vendor/akakooluo-customer-service-scripts/` — Reference library customer service scripts (MIT License, 75 file)

## Webhook Dedupe

Evolution retries webhook deliveries. **Always dedupe before inserting the inbound message and before generating a reply.**

Correct `main.py` sequence:
1. Extract message and upsert conversation.
2. SELECT `messages` WHERE `conversation_id` + `evolution_message_id` match.
3. If found → return `{"ok": True, "reply": "duplicate_ignored"}`.
4. Fetch recent history **before** inserting the new inbound message, so first-message greeting detection still sees an empty history.
5. Insert inbound message.
6. Generate reply, send via Evolution, then insert outbound message.

Critical pitfall: if you insert inbound first and then run the duplicate SELECT, every new message finds itself and Lia returns `duplicate_ignored` without ever sending a reply. Symptom pattern: DB has inbound messages but zero outbound replies, health check stays green, Evolution webhook appears enabled.

Add/keep regression tests for both cases:
- new message → inbound insert + sendText + outbound insert
- duplicate message → no insert, no sendText, no model call

Without dedupe, the same message creates 2-5 duplicate outbound replies; with dedupe in the wrong place, Lia goes silent.

## Read Receipts / Blue Ticks

Evolution `sendText` does not automatically mark the customer's inbound message as read. If Lia replies without marking the inbound message read, WhatsApp can show the customer's message as grey ticks even though Lia answered. For AI-handled messages, call:

```txt
POST /chat/markMessageAsRead/{instance}
{
  "readMessages": [
    {"remoteJid": "628xxx@s.whatsapp.net", "fromMe": false, "id": "<incoming_message_id>"}
  ]
}
```

Do this only when Lia is actually going to answer or handoff-answer. Avoid marking messages read while the conversation is paused for a human unless that is explicitly desired.

## Send Text Error Handling

Wrap `evolution.send_text()` in try/catch. If send fails (Evolution API down, network error):
- Log the error
- Don't crash the webhook handler
- The inbound message is already persisted; reply loss is acceptable

## Model Fallback Handling

`brain.generate_reply()` has a deterministic fallback when Hermes/model calls fail or return empty. That fallback must be context-aware:
- If `history` is empty, an intro/discovery fallback is OK.
- If `history` is non-empty, never ask name/business again; continue from context with a safe generic answer.
- Always log the model exception (`lia.brain`) before using fallback, otherwise repeated intro messages look like prompt bugs or restart-related memory loss.

Symptom: customer asks a follow-up like "Klo WA juga sama?" but Lia replies with first-message copy (`Selamat sore Kak, aku Lia... Boleh aku tahu nama...`). Root cause is usually model subprocess failure/timeout hitting the opening fallback, not lost DB memory.

## Message Buffering (WhatsApp-Native Interaction)

WhatsApp users often send multiple short messages in quick succession. Replying to each fragment individually feels robotic and can cause premature AI replies.

**Required behavior:**
- Buffer short successive user messages into a single turn
- Reset buffer timer on each new message
- Cap max wait (recommended: 3-5 seconds)
- Flush sooner for urgent/complaint messages
- Merge fragments into one context window before AI planning
- Never reply while user is still actively sending

See `references/state-machine-and-events.md` for the `user_buffering` state and buffer flush transitions.

## Outbox Pattern for Evolution Sends

Never call Evolution `sendText` directly inline in the webhook handler. Use an outbox pattern:

1. Generate AI reply text
2. Insert into `outbox_messages` table with status `pending`
3. Return webhook 200 immediately
4. Worker picks up pending outbox rows, calls Evolution `sendText`
5. On success → update status to `sent`, store `provider_message_id`
6. On failure → increment `attempt_count`, set `next_retry_at`

**Why:** This decouples webhook response from send latency, enables retry without regenerating AI replies, and provides the outbox ledger needed for `fromMe` classification (same-number human takeover).

See `references/evolution-same-number-handoff.md` for the outbox schema and `fromMe` classifier algorithm.

## Formal State Machine

The runtime must implement a formal state machine with audited transitions. See `references/state-machine-and-events.md` for the full 12-state model.

**Lifecycle states:**
`idle → user_buffering → ai_planning → ai_typing → ai_sending → idle` (normal flow)
`handoff_requested → waiting_human → human_active → resume_pending → idle` (handoff flow)
`failed_recoverable → retrying → idle|suspended` (failure recovery)

**Ownership states:** `ai | human | system`

Lifecycle state says *what is happening*. Ownership says *who is allowed to act*.

## Tool Safety

When tools/actions are added beyond simple text replies:

- Every tool must have a manifest with risk level, tenant capability check, and approval requirements
- Dangerous actions (discounts, bookings, invoices, refunds, customer-state mutations) require human approval unless tenant policy explicitly allows
- Tool calls must be idempotent with schema-validated inputs
- Results must be persisted and summarized
- Failures must have documented compensating actions

## Observability

The runtime must emit structured runtime events for:
- State transitions and ownership changes
- Model latency, cost, and errors
- Outbox send status and retry counts
- Duplicate webhook rate
- Handoff response time and human takeover duration
- Dead-letter review and replay

Alert on conversations stuck in `ai_planning`, `ai_sending`, `waiting_human`, or `failed_recoverable`.

## Runtime Capability Checklist

See `references/runtime-capability-checklist.md` for the full audit checklist covering: webhook ingestion, conversation processing, message buffering, human takeover, tenant isolation, orchestration, tool safety, and observability.

## Production Risks

See `references/production-risks.md` for the risk taxonomy (critical/high/medium) including: AI+human double-reply, duplicate webhook amplification, webhook blocking on model calls, `fromMe` classification failures, tenant data leaks, unapproved tool execution, and invalid model strings.

## Implementation Roadmap

See `references/implementation-roadmap.md` for the phased rollout order:
- Phase 0: Launch blockers (outbox, `fromMe` classifier, model validation)
- Phase 1: Runtime correctness (idempotency, queue, lock, state machine)
- Phase 2: WhatsApp-native interaction (buffering, pacing, chunking)
- Phase 3: Human collaboration (handoff sessions, admin release, continuity)
- Phase 4: Tenant reliability (RLS, capabilities, rate limits)
- Phase 5: Observability (timeline, dead letters, metrics)
- Phase 6: Multi-agent routing and approval-gated tools

## Dashboard-New-Platform Pattern

To add a new platform card to the dashboard homepage:
1. Add icon function in `src/app/page.tsx` (see `WhatsAppIcon` SVG pattern)
2. Add entry in `platforms` array with `href`, `label`, `icon`
3. Create `src/app/systems/<name>/page.tsx` + dashboard component
4. Copy from `/root/work/` to `/root/repos/`, then `npm run build && pm2 restart growthforge-mission-monitor --update-env`
