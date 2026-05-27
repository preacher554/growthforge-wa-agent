# Message buffer design

Indonesian WhatsApp users frequently send messages in rapid fragments before finishing their thought. The runtime must buffer these fragments, wait for a natural pause, then merge them into one coherent turn before AI planning begins.

## Why this matters

Without buffering:
- User sends "Kak" → AI plans and replies to a single-word greeting
- User sends "mau tanya soal paket" → AI plans and replies again
- User sends "yang pro itu gimana" → AI plans and replies a third time
- Customer receives 3 fragmented AI replies to one incomplete question

With buffering:
- All three fragments are held in the buffer
- After 1500ms of silence, they are merged into: "Kak mau tanya soal paket yang pro itu gimana"
- AI plans once, replies once, response is coherent

## Buffer window algorithm

```text
on message.received for conversation_id:

  1. Check for active buffer row in message_buffers where flushed_at IS NULL

  2a. If NO active buffer:
      INSERT into message_buffers:
        conversation_id, tenant_id
        messages = [current_message]
        flush_at = now() + buffer_window_ms
      Enqueue BullMQ delayed job:
        jobId = 'buffer_flush:{conversation_id}'
        delay = buffer_window_ms
      Emit: message.buffered
      Return

  2b. If ACTIVE buffer exists:
      APPEND current_message to buffer.messages
      UPDATE buffer.flush_at = now() + buffer_window_ms
      Cancel existing BullMQ delayed job by jobId
      Re-enqueue with new delay
      Emit: message.buffered
      Return

3. When BullMQ job fires (no new message arrived):
   UPDATE message_buffers SET flushed_at = now()
   Merge messages into one turn (see merge rules below)
   Emit: buffer.flushed
   Enqueue conversation for AI planning
```

## Timing values

| Parameter | Value | Notes |
|---|---|---|
| `buffer_window_ms` | 1500 | Reset on each new message |
| `max_wait_ms` | 8000 | Hard cap; flush even if messages keep arriving |
| `urgent_flush_ms` | 0 | Flush immediately on urgency signal |

The `max_wait_ms` cap prevents a chatty user from blocking the AI indefinitely.

## Fragment merge rules

```text
1. Sort messages by received_at ascending
2. Join with single space: " ".join(messages)
3. Preserve original capitalization and punctuation
4. Do not add conjunctions or filler words
5. Treat the merged result as a single inbound turn

Example:
  ["Kak", "mau tanya", "soal paket pro"] → "Kak mau tanya soal paket pro"
```

## Flush override conditions

Flush the buffer immediately and skip the remaining wait when:
- The buffered content contains a handoff trigger keyword (darurat, bayar, komplain, angry patterns)
- The buffer has exceeded `max_wait_ms`
- The conversation transitions to `handoff_requested` or `waiting_human` from another event

## Database table

```sql
CREATE TABLE message_buffers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id TEXT NOT NULL,
  conversation_id TEXT NOT NULL,
  messages JSONB NOT NULL DEFAULT '[]',
  -- each message: { id, content, received_at, raw_webhook_id }
  flush_at TIMESTAMPTZ NOT NULL,
  flushed_at TIMESTAMPTZ,
  merged_content TEXT,            -- set when flushed
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON message_buffers (conversation_id) WHERE flushed_at IS NULL;
```

## BullMQ job pattern

```text
Queue: message-buffer
Job ID: buffer_flush:{conversation_id}    // must be stable and unique per conversation

Using stable jobId enables:
  - cancel and re-add when timer resets (job.remove() then queue.add())
  - prevention of duplicate flush jobs
  - visibility in BullMQ dashboard for debugging

Never use setTimeout for buffer timers. Workers restart. setTimeout does not survive restarts.
```

## State machine integration

```text
conversation.state = 'user_buffering' while buffer window is open
conversation.state = 'ai_planning'   when buffer.flushed fires and AI call begins
```

Do not allow AI planning to start while state is `user_buffering`. If a new message arrives during `ai_planning`, it creates a new buffer for the next turn rather than interrupting the current one.

## Advanced: Redis ZSET sliding window (higher-precision alternative)

For deployments requiring sub-millisecond atomicity, or where BullMQ cancel/re-add latency causes rare duplicate flush races, use Redis Sorted Sets instead.

### Algorithm

```text
Buffer key:  buffer:msgs:{tenant_id}:{conversation_id}   (ZSET, score = execution_timestamp)
Timer key:   buffer:timer:{tenant_id}:{conversation_id}  (STRING)

On message.received (M, timestamp T):

  1. ZADD buffer:msgs:{key} T message_json
     (score = T; used for ordering, not flush timing)

  2. new_exec_time = T + buffer_window_ms
     SET buffer:timer:{key} new_exec_time

  3. Schedule flush check at new_exec_time (BullMQ delayed job or Redis keyspace notify)
     jobId = 'buf:{conversation_id}' (stable, cancels previous)

On flush job fires at T_exec:

  MULTI
    current_timer = GET buffer:timer:{key}
    IF current_timer != T_exec:
      DISCARD  // another message reset the window; this flush is stale
    ELSE:
      all_messages = ZRANGEBYSCORE buffer:msgs:{key} -inf +inf WITHSCORES
      DEL buffer:msgs:{key}
      DEL buffer:timer:{key}
  EXEC

  If EXEC returned messages:
    Sort by score (= original received_at)
    Merge into one turn
    Enqueue for ai_planning
```

### Why ZSET is more precise

| Concern | BullMQ delayed job | Redis ZSET |
|---|---|---|
| Timer reset race | Small window between remove() and add() | Atomic: timer comparison inside MULTI/EXEC |
| Survives worker restart | Yes (BullMQ persists) | Yes (Redis persists with AOF/RDB) |
| Message ordering | Relies on DB query ordering | ZSET score = received_at, deterministic |
| Implementation complexity | Lower — recommended for v1 | Higher — use when race conditions observed |

### Recommendation

Use BullMQ for v1. Switch to Redis ZSET if you observe double-flush events in production logs.

## Checklist

- [ ] Timer resets on every new message within the window
- [ ] Hard cap prevents indefinite blocking
- [ ] Stable BullMQ jobId prevents duplicate flush jobs
- [ ] Urgency keywords trigger immediate flush
- [ ] Merged content is stored in the buffer row for audit
- [ ] State transitions to user_buffering when buffer opens
- [ ] State transitions to ai_planning only after buffer flushed
