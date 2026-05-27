# Runtime capability checklist

Use this checklist when auditing a WhatsApp AI communication runtime.

## Webhook ingestion

- Validates webhook signature or shared secret when supported.
- Normalizes provider payloads into one internal event shape.
- Persists raw webhook payload before processing.
- Uses stable idempotency keys, for example `(provider, instance, message_id, event_type)`.
- Returns quickly and delegates work to a queue.
- Handles duplicate, late, replayed, and out-of-order events.
- Distinguishes one-to-one chats from groups, broadcasts, status updates, and unsupported channels.
- Does not block on model calls or Evolution API sends.
- Validates provider/model names before production deployment.

## Conversation processing

- Uses per-conversation FIFO queueing.
- Uses conversation-level locks or optimistic state versioning.
- Avoids concurrent AI replies for the same customer.
- Supports stale lock recovery.
- Stores inbound, outbound, system, tool, and human events.
- Keeps raw events immutable and derived state rebuildable.

## Message buffering

- Buffers short successive user messages.
- Resets timer on new message.
- Has a max wait cap.
- Flushes sooner for urgent or high-risk messages.
- Merges fragments into one turn before AI planning.
- Avoids replying while the user is still actively typing/sending.

## Human takeover

- Supports same-number human reply.
- Processes `fromMe` events instead of ignoring all of them.
- Maintains an outbound ledger to distinguish AI/system messages from human admin messages.
- Sets owner to `human` after unrecognized same-number outbound activity.
- Prevents AI from sending while human owns the conversation.
- Supports manual release and inactivity-based release.
- Writes a continuity summary before AI resumes.
- Provides a no-UI v1 release path, for example an admin private WhatsApp command.
- Audits handoff notification delivery and release reason.

## Tenant isolation

- Every table has `tenant_id` where data could cross tenant boundaries.
- Memory keys include tenant ID and customer ID.
- Vector search is tenant-scoped.
- Prompt/SOP/policy bundles are tenant-scoped.
- Tool permissions are tenant-scoped.
- Observability and billing are tenant-scoped.
- Admin users can access only authorized tenants.

## Orchestration

- Separates router, planner, memory builder, policy engine, tool executor, and sender.
- Uses model routing policies per tenant and task.
- Supports fallback models and timeouts.
- Does not regenerate a new answer when only message sending failed.
- Stores AI decisions before executing side effects.
- Uses delayed jobs for message buffering, pacing, and retry timing.

## Tool safety

- Tools have manifests.
- Inputs are schema-validated.
- Risk levels are explicit.
- Dangerous actions require approval.
- Tool calls are idempotent.
- Results are persisted and summarized.
- Failures are recoverable or compensating actions are documented.

## Observability

- Emits structured runtime events.
- Tracks state transitions.
- Tracks ownership changes.
- Tracks model latency, cost, and errors.
- Tracks outbox send status.
- Tracks duplicate webhooks and retry counts.
- Has dead-letter review and replay.
- Has an admin timeline for debugging a single conversation.
- Alerts on conversations stuck in `ai_planning`, `ai_sending`, `waiting_human`, or `failed_recoverable`.

## Humanization and pacing

- Response delay is calculated from message length, not a constant.
- Urgency signals shorten the delay; frustrated or angry signals add a small increase.
- Evolution sendPresence composing is called before every outbound message.
- Long messages over 280 characters are split at natural paragraph breaks.
- Inter-chunk delays include typing indicator between each chunk.
- Pacing waits use BullMQ delayed jobs, not setTimeout.
- Minimum inter-message gap is enforced across chunks.

## Emotional and urgency signals

- Signals are written to the conversation_signals table, not stored only in prompts.
- LLM outputs a structured signal block alongside every reply.
- Rule-based keyword detection fires on message receipt before LLM call for urgency cases.
- Urgency signal triggers immediate message buffer flush.
- Escalation threshold check runs after every signal write.
- Critical escalation auto-triggers handoff without waiting for the next AI turn.

## Continuity on AI resume

- Continuity summarizer runs before every resume_pending to idle transition.
- Human session messages are collected from the messages table.
- Summary is injected as a labeled block in the AI context window.
- Customer memory is namespaced by tenant_id and customer_jid, never phone number alone.
- Context assembly follows defined layer order: system prompt, tenant knowledge, customer memory, continuity summary, recent history, current turn.
- Session summaries are written at session end to prevent context window overflow.

## Circuit breaker and resilience

- LLM failures use exponential backoff, not immediate re-queue.
- Retry exhaustion auto-escalates conversation to human ownership.
- Customer receives an apologetic message when AI fails; never left in silence.
- Admin receives SOS notification when system auto-escalates.
- Outbox retries resend from stored outbox content, never regenerate from LLM.
- Stale lock recovery worker runs on schedule (every 5 minutes or equivalent).
- Circuit breaker state tracked in Redis per provider for scale deployments.
- Dead-letter rows contain enough payload for manual replay and debugging.

## Cooldown and silent observer

- COOLDOWN state is implemented between continuity summary write and AI resume.
- Cooldown window is 30 seconds; a new human message during cooldown returns to human_active.
- During human_active, resume_pending, and cooldown: AI workers may update memory but must not write to outbox or trigger sends.
- Silent observer behavior is enforced at the outbox writer level, not only in the planning prompt.

## Observability: time-travel and replay

- runtime_events has a per-conversation sequence number column.
- Conversation state can be reconstructed to any sequence point by replaying events in order.
- Admin timeline query shows all state transitions, ownership changes, and outbox events for a single conversation.
- Duplicate webhook rate is tracked and alerted when it exceeds threshold.
- LLM call latency and cost are tracked per tenant per model.
