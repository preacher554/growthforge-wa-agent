# Implementation roadmap

Use this order when turning an MVP WhatsApp bot into a production runtime. Do not start with multi-agent routing before correctness and ownership are stable.

## Phase 0 - Launch blockers

1. Implement outbox table and sender worker.
2. Implement `fromMe` classifier using the outbox ledger.
3. Prove same-number human reply transitions the conversation to `human_active`.
4. Prove AI/system outbound confirmations do not trigger human takeover.
5. Validate actual model provider and model string.

## Phase 1 - Runtime correctness

1. Add webhook idempotency table with composite unique key.
2. Make webhook handler return 200 quickly after persistence and enqueue.
3. Add per-conversation FIFO queue.
4. Add conversation lock with TTL or optimistic versioning.
5. Add formal lifecycle state machine and ownership state.
6. Add insert-only runtime event log.
7. Add outbox retry without regenerating AI replies.

## Phase 2 - WhatsApp-native interaction

1. Add message buffer with reset timer and max wait.
2. Merge fragments before AI planning.
3. Add typing indicator/presence before sends.
4. Add response delay calculation.
5. Add long-message chunking with inter-chunk delay.
6. Add urgency and complaint overrides.

## Phase 3 - Human collaboration

1. Add handoff sessions table.
2. Add admin private WhatsApp notification audit.
3. Add no-UI admin release command.
4. Add `resume_pending` continuity summarizer.
5. Inject human-session summary into next AI context.
6. Add stale handoff and stale lock recovery.

## Phase 4 - Tenant reliability

1. Enforce `tenant_id` on every tenant-scoped table.
2. Add Supabase/Postgres RLS when any admin UI or tenant API exists.
3. Add tenant capability checker.
4. Add model routing and cost limits per tenant.
5. Add rate limits and usage events.

## Phase 5 - Observability

1. Build admin conversation timeline from runtime events.
2. Add dead-letter review and replay.
3. Track duplicate webhook rate.
4. Track LLM latency, send latency, and cost.
5. Track handoff response time and human takeover duration.
6. Track emotional and lead signals per tenant.

## Phase 6 - Multi-agent and tools

1. Add internal router for sales, support, booking, escalation, and analytics.
2. Add tool manifest format.
3. Add policy gate and approval flow.
4. Add idempotency and audit for every tool execution.
5. Add tenant package boundaries for commerce, booking, invoices, payments, and integrations.
