# Runtime redesign report

## Executive verdict

State whether the current system is a prompt bot, MVP webhook, early runtime, or production-grade communication runtime. Be direct.

If the repository is documentation/SOP/prompt/YAML only, say it is not the executable runtime. Separate "audited code" findings from "described architecture" findings.

## Deployment blockers

List anything that must be fixed before real customers use the system. Always check:

- Are all `fromMe=true` events ignored?
- Is there an outbox ledger?
- Can same-number human replies be detected?
- Does webhook processing block on LLM or send calls?
- Is webhook idempotency enforced before queueing?
- Is the configured model string valid?

## Current architecture

- Repository/runtime components inspected:
- Message path:
- Data stores:
- Model routing:
- WhatsApp bridge:
- Tenant model:
- Handoff model:

## Critical weaknesses

List the most severe runtime risks first. Include file references when code is available.

If same-number handoff is a product requirement and `fromMe=true` messages are ignored, mark it as critical and deployment-blocking.

## Missing runtime systems

- Event log:
- Queue:
- Idempotency:
- Conversation lock:
- Message buffer:
- Ownership engine:
- Outbox:
- Policy engine:
- Memory sync:
- Observability:
- Failure recovery:

## Target architecture

```text
WhatsApp bridge
  -> Ingestion gateway
  -> Event log
  -> Conversation queue
  -> Message buffer
  -> Ownership/state machine
  -> Router/orchestrator
  -> Policy/tool gate
  -> Outbox sender
  -> Observability/recovery
```

## State machine

Define lifecycle states, ownership states, and transition events. Prefer a lifecycle similar to:

```text
idle
user_buffering
ai_planning
ai_typing
ai_sending
handoff_requested
waiting_human
human_active
resume_pending
failed_recoverable
retrying
suspended
archived
```

## Human-AI collaboration

Explain same-number takeover, admin detection, AI pause, manual release, inactivity unlock, and continuity after human messages. Include the outbox-based `fromMe` classifier.

## Multi-tenant strategy

Explain tenant isolation across data, memory, prompts, tools, logs, model routing, and WhatsApp sessions.

## Memory strategy

Describe event memory, conversation memory, customer memory, business knowledge, and human takeover continuity summaries.

## Tool safety

Define risk levels, approvals, tool manifests, idempotency, validation, and audit.

## Observability

Define events, logs, metrics, traces, admin timeline, dead letters, and replay.

## Humanization and interaction quality

Assess whether the runtime produces natural-feeling WhatsApp interactions.

- Is response delay calculated or constant?
- Is a typing indicator shown before sending?
- Are long messages chunked at paragraph boundaries?
- Is pacing implemented as a durable delayed job or fragile setTimeout?
- What does the reply timing look like to a customer?

Describe the humanization gap and what the target behavior is.

## Emotional signal layer

Assess whether emotional and urgency signals are tracked as runtime data.

- Are signals written to a database table or only used inside prompts?
- Can escalation routing use accumulated signal history?
- Does urgency detection short-circuit the buffer window?
- Can the tenant see emotional state analytics?

Describe what needs to be added and at which phase.

## Competitive differentiation opportunities

Identify specific runtime behaviors where this system can exceed commodity WhatsApp AI platforms.

Common areas where managed WhatsApp AI products are weak:
- Fragment handling: reply to each fragment individually instead of merging
- Typing pacing: instant replies that feel robotic
- Same-number takeover: route customer to a different number or no detection at all
- Context after human session: AI resumes with no knowledge of what the human said
- Emotional routing: no threshold logic, just prompt instructions

List which of these are addressed by the current design and which remain as gaps.

## Resilience and failure behavior

Assess whether the runtime handles failures without leaving customers in silence.

- What happens when the LLM provider times out or returns 429?
- Is retry backoff exponential or immediate?
- What happens when retries are exhausted?
- Does the customer receive a message? Does the admin receive an alert?
- Can outbox retries regenerate a different AI reply than the original?
- Is there a stale lock recovery mechanism?

If any of these produce silence or undefined behavior, list them as High or Critical risks.

## Technology recommendations

Recommend concrete infrastructure based on the user stack. Include migration-safe alternatives.

For Node.js + Evolution API runtimes, BullMQ + Redis is the practical v1 recommendation. For 50+ concurrent active conversations, note that Kafka partition-by-phone provides stronger FIFO ordering guarantees.

## Implementation roadmap

1. Phase 0: launch blockers, especially outbox, `fromMe` classifier, same-number handoff proof, and model validation.
2. Phase 1: correctness foundation with idempotency, fast webhook return, queue, lock, state machine, runtime events, and outbox retry.
3. Phase 2: WhatsApp-native interaction with buffering, typing indicator, pacing, chunking, and urgency overrides.
4. Phase 3: human collaboration with handoff sessions, admin release command, continuity summary, and stale recovery.
5. Phase 4: tenant reliability with RLS, capabilities, model routing, rate limits, and usage.
6. Phase 5: observability with timeline, dead letters, latency, cost, duplicate rate, and emotional analytics.
7. Phase 6: multi-agent routing and approval-gated tools.
