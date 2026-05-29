# GrowthForge WA Agent Architecture Playbook v2

> Reverse-engineered from Evolution API, LangGraph WhatsApp agent patterns, ecommerce WhatsApp sales bot patterns, and Wassenger's multimodal WhatsApp AI bot.

## Goal

Build an AI workforce layer for Indonesian businesses, starting with WhatsApp-only 1-on-1 AI receptionist + light sales.

GrowthForge is not just building a chatbot. GrowthForge is building shared AI operations infrastructure where each client gets its own AI context, SOP, memory, WhatsApp instance, products/SKU, client records, and handoff rules while sharing the same core runtime.

## Current GrowthForge baseline

```txt
Hermes/Yuya        = operator/orchestration brain
Evolution API     = WhatsApp bridge / transport layer
Supabase          = memory + CRM + analytics database
Hermes OpenAI OAuth = Lia model path for sandbox/runtime brain
OpenRouter        = future production worker/model routing layer when separated by worker profile
```

Current runtime:

```txt
WhatsApp customer
↓
Evolution instance: lia-growthforge
↓
Webhook: MESSAGES_UPSERT
↓
FastAPI Lia runtime
↓
Supabase persistence
↓
Hermes/OpenAI OAuth reply generation
↓
Evolution sendText
```

## Key lessons from reference repos

### 1. Evolution API

Evolution is a multi-instance WhatsApp gateway, not the business brain.

Useful patterns:

- Instances are the natural tenant boundary at the transport layer.
- Important endpoints include:
  - `POST /instance/create`
  - `GET /instance/connect/:instanceName`
  - `GET /instance/connectionState/:instanceName`
  - `DELETE /instance/logout/:instanceName`
  - `DELETE /instance/delete/:instanceName`
  - `POST /webhook/set/:instanceName`
  - `GET /webhook/find/:instanceName`
  - `POST /message/sendText/:instanceName`
- Webhook config is wrapped under top-level `webhook` in v2.3.x:

```json
{
  "webhook": {
    "enabled": true,
    "url": "http://host.docker.internal:3300/webhook/evolution",
    "events": ["MESSAGES_UPSERT", "CONNECTION_UPDATE"],
    "byEvents": false,
    "base64": false
  }
}
```

GrowthForge takeaways:

- Treat `instanceName` as the WhatsApp channel identifier.
- Map `instanceName → tenant` in Supabase.
- Never let raw webhook events directly decide business behavior without normalization, dedupe, and state checks.
- For Docker Evolution → host runtime, add `host.docker.internal:host-gateway` and verify `/health` from inside the Evolution container.
- Expect retries. Webhook handlers must be idempotent.

### 2. LangGraph WhatsApp Agent

Useful patterns from the LangGraph repo:

- Conversation memory is checkpointed by `thread_id` derived from phone number.
- State graph is simple at first: `START → assistant → END`.
- PostgreSQL checkpointer persists conversation state.
- Message aggregation waits briefly before processing multiple quick user messages.
- Voice note support is handled before the agent step: transcribe audio → feed text into agent.

GrowthForge takeaways:

- Use `client_id = remote_jid` as the stable thread key.
- Add a short debounce/aggregation window before AI response to avoid replying three times if the end user sends fragmented messages.
- Store both normalized memory and raw events.
- Keep the graph small in MVP, but design for nodes later:

```txt
normalize_event
↓
load_tenant_context
↓
load_client_memory
↓
classify_intent
↓
sales_receptionist_reply / tool_call / escalate
↓
persist_outcome
↓
send_reply
```

### 3. WhatsApp ecommerce sales bot

Useful patterns:

- It uses explicit stages:
  - welcome
  - menu
  - address
  - bill/order summary
  - new order
  - assistant/human
  - abandoned cart recovery
- User state is stored per phone number.
- Products are a simple menu/SKU object.
- Human handoff can be triggered via option `0`.
- Abandoned cart recovery scans stale stage states and sends reminders.

GrowthForge takeaways:

- Even with LLMs, use explicit state for sales-critical flows.
- Product/SKU handling should be deterministic, not purely prompt-based.
- Store carts/orders/leads as structured objects.
- Recovery/follow-up is a separate workflow, not part of the immediate reply loop.

Recommended state model:

```txt
new_lead
collecting_identity
collecting_need
qualifying
recommending_offer
waiting_human
human_active
follow_up_due
resolved
```

### 4. Wassenger multimodal AI bot

Useful patterns:

- Eligibility filter before replying:
  - skip groups
  - skip own messages
  - skip assigned-to-human chats
  - skip blacklisted numbers/labels
  - skip archived/blocked chats
- Chat history is pulled and capped to recent messages.
- Human command triggers assignment/escalation.
- Tool/function calling is used for:
  - plan/pricing retrieval
  - CRM lookup
  - meeting availability
  - booking meeting
  - current date/time
- Multimodal support is modular:
  - audio input transcription
  - audio output
  - image input
- Quota/rate limits prevent infinite chat loops and runaway cost.

GrowthForge takeaways:

- Add `can_reply()` before any model call.
- Add per-tenant quotas.
- Make tool calls first-class:

```txt
get_products
get_pricing_scope
save_lead_profile
update_client_memory
create_handoff
schedule_follow_up
check_meeting_slot
create_invoice_draft
```

## Target multi-tenant architecture

### Shared infrastructure

```txt
Evolution API cluster
Supabase project/database
Hermes/Yuya orchestration
Runtime API service
Worker/model routing layer
Analytics/dashboard layer
```

### Per tenant/client data

Each client has:

```txt
tenant row
WhatsApp instance
AI persona/context
SOP documents
product/SKU catalog
client records
conversation state
memory namespace
handoff rules
billing/onboarding status
analytics namespace
```

Recommended tenant key format:

```txt
<client_slug>_<agent_slug>
```

Examples:

```txt
growthforge_lia
clinicabc_reva
restobandung_nara
```

## Supabase schema v2 recommendation

Existing tables:

```txt
tenants
conversations
messages
handoff_events
lead_summaries
```

Add next:

```txt
tenant_sops
agent_profiles
client_profiles
products
product_variants
conversation_memory
conversation_events
sales_opportunities
follow_up_tasks
tool_calls
agent_metrics_daily
```

### Customer ID standard

Primary client identity:

```txt
remote_jid = WhatsApp JID, e.g. 6281234567890@s.whatsapp.net
```

Recommended normalized fields:

```txt
remote_jid          -- platform-stable WhatsApp ID
phone_e164          -- +6281234567890
display_name        -- WhatsApp push name or collected name
collected_name      -- name given by customer
business_name       -- client's business/brand
source_channel      -- whatsapp
```

## Sales receptionist flow v1

The assistant should not jump straight into explanation. Opening flow:

```txt
1. Greeting
2. Introduce self
3. Ask end user name
4. Ask business/brand type
5. Ask what they want help with
6. Qualify lightly
7. Recommend product path
8. Escalate if custom/high intent
```

Example opening:

```txt
Halo Kak, kenalin aku Lia dari GrowthForge. Aku bantu arahkan kebutuhan AI/otomasi bisnis Kakak ya. Boleh aku tahu nama Kakak dan bisnis/brand Kakak bergerak di bidang apa?
```

Discovery questions, one or two at a time:

```txt
- Saat ini paling banyak kendalanya di balas chat, follow-up lead, atau bikin konten/growth?
- Dalam sehari kira-kira ada berapa chat/lead masuk?
- Sudah ada admin manusia atau masih dipegang owner langsung?
- Target Kakak sekarang lebih ke hemat waktu, naik closing, atau rapiin operasional?
```

## Escalation logic

Escalate when:

```txt
custom pricing
integration request
payment/contract/legal/refund/complaint
high-value sales intent
end user asks human/admin/meeting
AI uncertainty high
end user repeats same question after failed answer
conversation exceeds quota
```

On escalation:

```txt
1. Save handoff_event
2. Set conversation.state = waiting_human
3. Notify admin/private channel
4. Stop AI auto-reply for that end user
5. Human replies from same WhatsApp number
6. Human can resume AI later
```

## Memory architecture

Use three layers:

### 1. Message log

Raw transcript and audit trail.

```txt
messages
```

### 2. Conversation state

Operational state.

```txt
conversations.state
sales_opportunities.stage
follow_up_tasks.status
```

### 3. Semantic/client memory

Short structured facts.

```txt
client_profiles
conversation_memory
lead_summaries
```

Examples:

```txt
collected_name: Budi
business_type: dental clinic
need: WA sales receptionist
pain: slow response after work hours
lead_temperature: warm
preferred_contact_time: evening WIB
```

Do not rely only on prompt history. Store structured fields.

## Tool-calling roadmap

Phase 1 tools:

```txt
save_client_profile
save_lead_summary
create_handoff
get_product_catalog
get_package_scope
```

Phase 2 tools:

```txt
check_meeting_slot
book_consultation
create_follow_up_task
create_invoice_draft
lookup_customer_history
```

Phase 3 tools:

```txt
check_order_status
calculate_shipping
check_inventory
update_crm_stage
trigger_campaign
```

## Product/SKU architecture

For each tenant:

```txt
products
- id
- tenant_id
- sku
- name
- description
- category
- base_price
- currency
- active
- metadata jsonb
```

For GrowthForge itself, initial products:

```txt
WA Agent Basic
WA Agent Pro
InstaGrow Ops
Custom AI Ops System
```

Rule:

- AI may explain scope and fit.
- AI must not invent final custom price.
- AI can collect requirements and escalate.

## Analytics v1

Track:

```txt
total_customers
new_customers_today
messages_inbound
messages_outbound
avg_first_response_time
handoff_count
lead_temperature_count
conversation_state_count
top_intents
```

## Implementation priorities

### Immediate MVP fixes

- Opening flow: introduce Lia + ask name/business.
- Add customer profile table.
- Extract and store collected name/business.
- Add conversation state stages beyond `ai_active`.
- Add admin handoff notification.

### Next production step

- Add message aggregation/debounce.
- Add `can_reply()` eligibility filter.
- Add tool-call layer.
- Add product catalog tables.
- Add lead summary generator.
- Add dashboard queries.

### Later

- Voice note transcription.
- Image understanding.
- Omnichannel abstraction.
- Official WhatsApp Cloud API option for high-compliance clients.
- Multi-worker operations with GrowthForge profiles.

## Guardrails

- WhatsApp gateway is transport, not truth.
- Supabase is source of operational truth.
- AI should never own billing/legal/final pricing without approval.
- Human handoff must pause AI for that conversation.
- Every incoming event must be idempotent.
- Tenant boundary must be enforced on every query.
- Do not mix client memory/SOP/product catalog across tenants.
