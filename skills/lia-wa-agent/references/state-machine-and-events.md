# State machine and event model

Use this as a starting point. Adjust names to the codebase, but preserve the separation between lifecycle, ownership, and delivery.

## Conversation lifecycle states

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

## Ownership states

```text
owner = ai | human | system
```

Lifecycle state says what is happening. Ownership says who is allowed to act.

## Core transitions

```text
idle + message.received -> user_buffering
user_buffering + buffer.flush_due -> ai_planning
ai_planning + ai.decision_ready -> ai_typing
ai_typing + pacing.elapsed -> ai_sending
ai_sending + outbox.sent -> idle

handoff.requested -> waiting_human
human.outbound_detected -> human_active
human.inactivity_timeout -> resume_pending
admin.release_requested -> resume_pending
continuity.summary_written -> idle

send.failed -> failed_recoverable
retry.scheduled -> retrying
retry.succeeded -> idle
retry.exhausted -> suspended
```

## Required runtime events

```text
webhook.received
webhook.duplicate_ignored
message.normalized
message.persisted
message.buffered
buffer.flushed
ownership.lock_acquired
ownership.lock_rejected
ownership.changed
handoff.requested
handoff.notified
human.outbound_detected
ai.plan_started
ai.plan_completed
ai.plan_failed
tool.proposed
tool.approval_requested
tool.approved
tool.rejected
tool.executed
outbox.created
outbox.send_started
outbox.sent
outbox.failed
state.transitioned
memory.summary_updated
deadletter.created
```

## Minimal table concepts

```text
runtime_events
conversation_ownership
message_buffers
outbox_messages
handoff_sessions
ai_decisions
tool_calls
tool_approvals
conversation_summaries
customer_memories
dead_letters
```

## Same-number human detection

When Evolution or another WhatsApp bridge emits `fromMe=true`:

1. Check whether the message matches a runtime outbox row by provider message ID, request ID, or text hash/time window.
2. If matched, classify as `ai` or `system` outbound.
3. If not matched, classify as human admin outbound.
4. Set ownership to `human`.
5. Record `last_human_activity_at`.
6. Keep AI silent until manual release or inactivity policy.

Never implement same-number takeover by dropping every `fromMe=true` event. That prevents the runtime from detecting human admin activity.
