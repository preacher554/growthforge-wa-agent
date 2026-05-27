# Lia Runtime vs WA Agent Architect V4 Audit

Date: 2026-05-27  
Auditor: Yuya  
Scope: `/root/repos/whatsapp-agent-architect` master skill + `/root/repos/WA_Agent_Persona_Lia` persona + live runtime `/root/repos/growthforge-wa-agent-runtime`

## Executive verdict

Lia is configured and behaving as a **Pro / AI Sales Receptionist** agent, not Basic.

However, Lia is **not yet fully V4-complete**. She is closer to a **V3.5 / Pro MVP**:

- Pro persona and package scope are present.
- Runtime core safety exists for the most critical WhatsApp flow.
- Same-number human takeover exists in a pragmatic form.
- But V4 structural requirements like outbox, message buffering, full formal lifecycle state machine, tool manifest layer, observability/dead letters, and expanded v2 schema are not fully implemented yet.

## Package classification

### Evidence that Lia is Pro

Persona repo:

```yaml
tenant_id: growthforge_lia
agent_role: whatsapp_frontdesk_sales
package: pro
skills:
  - wa-agent-core
  - wa-agent-basic
  - wa-agent-pro
capabilities:
  lead_qualification: true
  objection_handling: true
  lead_summary: true
  human_handoff: true
```

Runtime prompt (`app/brain.py`) explicitly says:

```txt
Kamu adalah Lia, Pro AI WhatsApp Sales Receptionist Nusavox.
```

Runtime prompt includes Pro behavior:

- sales discovery ringan
- pain point / urgency / budget signal
- lead qualification hot/warm/cold
- recommended next action
- Basic vs Pro package recommendation
- Custom/add-on scope wall

Database migration seeds the tenant as:

```sql
insert into tenants (... package ...)
values (... 'pro' ...)
```

### Conclusion

Lia is **Pro** by config, prompt, tenant metadata, and intended behavior.

## V4 alignment checklist

### Runtime Core

- FastAPI webhook + Evolution bridge: **Implemented**
  - `app/main.py` exposes `/webhook/evolution`
  - `app/evolution_client.py` sends via Evolution API

- Supabase/Postgres persistence: **Implemented, v1 schema**
  - tables: `tenants`, `conversations`, `messages`, `handoff_events`, `lead_summaries`
  - v2/V4 tables like `customer_profiles`, `conversation_memory`, `conversation_events`, `outbox_messages`, `tool_calls`, `dead_letters` are not yet present.

- Multi-tenant lookup by instance: **Implemented basic form**
  - `Store.get_tenant_by_instance(instance_name)`
  - good enough for one/multiple instances, but no full tenant capability gating yet.

- Webhook dedupe before insert: **Implemented**
  - `message_exists()` runs before `insert_message()`.
  - This fixes the previous silent duplicate bug pattern.

- Read receipts: **Implemented**
  - `mark_message_as_read()` exists and is called only when Lia will answer or handoff-answer.

- Kill switch: **Implemented service-level**
  - `WA_AGENTS_ENABLED` gate exists.
  - Note: direct `load_settings()` outside systemd may show default false because `WA_AGENTS_ENABLED=true` is injected by `lia-runtime.service` environment, while the live `/health` correctly reports true.

- Model routing via config: **Implemented**
  - `hermes_model_provider`, `hermes_model`, `hermes_timeout_seconds`.
  - Current source defaults are `openrouter / owl-alpha`; service env can override if configured.

- Model fallback: **Implemented**
  - context-aware fallback in `brain.py` avoids restarting opening flow when history exists.

- Message extraction: **Partially implemented**
  - supports `conversation`, `extendedTextMessage`, image/video captions, button/list response text.
  - does not yet handle audio transcription or richer attachment workflows.

- Session context labeling: **Implemented basic form**
  - history labels Customer/Lia/Admin GrowthForge based on direction and `fromMe` raw flag.

- Human handoff state protection: **Implemented**
  - `waiting_human` / `human_active` pause logic exists.

- Same-number takeover protection: **Implemented basic form**
  - `fromMe:true` is persisted and changes paused conversations to `human_active`.
  - Limitation: no outbox ledger classifier yet, so AI-sent messages vs human-sent messages are not robustly distinguished beyond runtime-generated synthetic message IDs and raw payload shape.

- Manual or timeout resume: **Implemented**
  - admin commands `/resume`, `/lanjut`, `/release`, `/ai`
  - 1-hour resume window for `human_active` and conservative fallback for `waiting_human`.

### Pro Scope

- Time-aware greeting: **Implemented**
- Product explanation before needs: **Implemented in prompt/opening**
- Ask customer name + business: **Implemented in opening**
- Sales discovery: **Implemented in prompt, not structured extraction**
- Pain point / urgency / budget signal: **Prompt-level only**
- Lead scoring hot/warm/cold: **Prompt-level + schema table exists, but no structured classifier/upsert flow yet**
- Objection handling: **Prompt-level only; no approved objection library enforcement**
- Full lead summary: **Schema exists, but runtime does not automatically generate/upsert full lead summary yet**
- Recommended next action: **Prompt-level/admin notification partially**
- Light follow-up: **Not implemented as scheduler/task**

### V4 structural requirements

- Formal 12-state lifecycle machine: **Not fully implemented**
  - Current states: `ai_active`, `waiting_human`, `human_active`, `resolved`.
  - V4 target includes `idle`, `user_buffering`, `ai_planning`, `ai_typing`, `ai_sending`, `handoff_requested`, `resume_pending`, `failed_recoverable`, `retrying`, `suspended`, etc.

- Message buffering/debounce: **Not implemented**
  - Runtime replies per incoming webhook message.
  - V4 wants 3-5 second buffer for fragmented WhatsApp messages.

- Outbox pattern: **Not implemented**
  - Runtime currently calls `send_text()` inline in the webhook handler.
  - V4 requires `outbox_messages` pending → worker send → status update.

- Robust fromMe classifier using outbox: **Not implemented**
  - Basic `fromMe:true` handling exists.
  - V4 wants classifier to distinguish AI outbox sends from same-number human/admin sends.

- Tool manifest and approval gates: **Not implemented**
  - No tool manifest layer yet.
  - Good: dangerous scopes are blocked by prompt/policy, but runtime does not enforce per-tool risk.

- Observability/dead-letter/replay: **Not implemented beyond logs**
  - no `runtime_events`, `dead_letters`, outbox retry metrics, or stuck-state alerts yet.

- Capability gating from tenant.yaml: **Not wired into runtime yet**
  - persona repo has `tenant.yaml` package/capabilities.
  - runtime currently uses database tenant + prompt, not a loaded YAML capability manifest.

## Important inconsistencies to clean up

1. **Runtime path naming drift**
   - Master skill says runtime path: `/root/repos/whatsapp-agent-architect-runtime`
   - Live service actually runs: `/root/repos/growthforge-wa-agent-runtime`
   - Current systemd unit confirms the live path.

2. **Brand drift: GrowthForge vs Nusavox**
   - Master repo name/readme says GrowthForge WA Agent.
   - Persona and runtime prompt say Nusavox.
   - Migration seeds business name as `GrowthForge`.
   - This may be intentional product-brand separation, but Nara should not assume it is aligned. Decide one source of truth:
     - GrowthForge as parent/operator and Nusavox as customer-facing WA Agent brand, or
     - migrate all customer-facing copy back to GrowthForge.

3. **Model documentation drift**
   - Some skill docs mention old/current models inconsistently.
   - Always verify live `app/config.py` and service env before diagnosing.

4. **V4 docs are ahead of runtime**
   - This is okay if treated as roadmap.
   - It is unsafe if Nara reports “V4 fully implemented.”

## Verification result

Live read-only checks:

```txt
lia-runtime.service: active + enabled
GET http://127.0.0.1:3300/health:
  ok=true
  service=lia-runtime
  instance=lia-growthforge
  wa_agents_enabled=true
Evolution key drift check:
  keys_match=true
  container_len=50
  env_len=50
  prefix=gf_e***
Evolution instance state:
  lia-growthforge=open
Evolution webhook:
  enabled=true
  url=http://host.docker.internal:3300/webhook/evolution
  events=[MESSAGES_UPSERT, SEND_MESSAGE]
```

Runtime test suite using the service venv:

```txt
/root/repos/growthforge-wa-agent-runtime/.venv/bin/python -m pytest -q
Result: 16 passed, 2 failed
```

The 2 failures are stale test expectations after the GrowthForge → Nusavox customer-facing rename:

```txt
tests/test_brain.py::test_opening_greeting_introduces_lia_and_asks_name_without_model_call
- expected: GrowthForge in deterministic opening
- actual: Nusavox in deterministic opening

tests/test_brain.py::test_opening_prompt_requires_receptionist_discovery_flow
- expected exact old phrase: "kenalkan diri sebagai Lia"
- actual prompt uses equivalent instruction: "Kenalkan diri sebagai Lia dari Nusavox"
```

Interpretation: these are **test-maintenance failures**, not evidence that Lia is Basic or that webhook/runtime is down. Runtime behavior should still be reviewed for brand source-of-truth: GrowthForge as parent/operator vs Nusavox as customer-facing WA Agent brand.

## Final classification

```txt
Package: Pro
Runtime maturity: Pro MVP / V3.5
V4 status: Partially aligned, not fully implemented
Production posture: live and reachable for controlled inbound testing, not yet full V4 hardened runtime
Verification caveat: pytest has 2 stale brand/prompt expectation failures after Nusavox rename
```

## Recommended next implementation order

1. **Outbox + send worker**
   - Add `outbox_messages` table.
   - Webhook inserts pending outbound, returns 200.
   - Worker sends through Evolution and updates status.
   - Enables robust AI-vs-human `fromMe` classification.

2. **Formal state machine v4-lite**
   - Expand states or add separate lifecycle/ownership columns.
   - Preserve existing `ai_active`, `waiting_human`, `human_active` mapping during migration.

3. **Message buffer/debounce**
   - Add short buffer for rapid multi-message WhatsApp input.
   - Prevent robotic premature replies.

4. **Structured lead profile + summary upsert**
   - Extract name, business, need, pain, urgency, budget_signal, service_interest, lead_status.
   - Populate `lead_summaries` consistently.

5. **Capability manifest loader**
   - Runtime should load tenant/package capabilities from DB or `tenant.yaml`-derived config.
   - Enforce Basic/Pro/Custom boundaries outside prompt text.

6. **Observability + repair runbooks**
   - Runtime events, send failures, duplicate counts, stuck conversations, webhook status.

## Troubleshooting SOP added

Added:

```txt
runbook/evolution-key-webhook-troubleshooting.md
```

It documents the 2026-05-27 incident where:

- `.env` Evolution API key was stale
- live container key differed
- protected endpoints returned `401`
- webhook URL was correct but disabled
- repair required syncing key, restarting only Lia, enabling webhook events `MESSAGES_UPSERT` + `SEND_MESSAGE`, then verifying logs
