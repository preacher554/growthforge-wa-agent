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
