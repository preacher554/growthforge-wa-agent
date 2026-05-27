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

## Technology recommendations

Recommend concrete infrastructure based on the user stack. Include migration-safe alternatives.

For Node.js + Evolution API runtimes, BullMQ + Redis is usually the practical v1 recommendation for queues, delayed buffer flushes, pacing jobs, retries, and dead-letter handling.

## Implementation roadmap

1. Phase 0: launch blockers, especially outbox, `fromMe` classifier, same-number handoff proof, and model validation.
2. Phase 1: correctness foundation with idempotency, fast webhook return, queue, lock, state machine, runtime events, and outbox retry.
3. Phase 2: WhatsApp-native interaction with buffering, typing indicator, pacing, chunking, and urgency overrides.
4. Phase 3: human collaboration with handoff sessions, admin release command, continuity summary, and stale recovery.
5. Phase 4: tenant reliability with RLS, capabilities, model routing, rate limits, and usage.
6. Phase 5: observability with timeline, dead letters, latency, cost, duplicate rate, and emotional analytics.
7. Phase 6: multi-agent routing and approval-gated tools.
