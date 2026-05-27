# Humanization and pacing engine

WhatsApp users notice inhuman timing immediately. A reply arriving within one second marks the interaction as a bot and reduces trust and compliance, regardless of how good the content is. This reference defines the runtime layers required to make AI responses feel like they came from a real person.

## Layer 1 — Response delay calculation

Calculate a pacing delay before sending every outbound message. Do not use a fixed constant.

### Mathematical formula

```
T_delay = T_base + (N_char / WPM_char) × 60

Where:
  T_delay   = total delay in seconds before message is sent
  T_base    = perceived read time for the incoming message (seconds)
  N_char    = character count of the AI response being sent
  WPM_char  = simulated human typing speed = 250 characters per minute

Example:
  Response of 180 characters, T_base = 2.0s:
  T_delay = 2.0 + (180 / 250) × 60 = 2.0 + 43.2 = 45.2s → capped at 8s
  T_delay = min(T_delay_calculated, T_max) = 8s
```

Cap: `T_max = 8.0 seconds`. Never delay longer than this regardless of response length.

### T_base: sentiment-aware read time

`T_base` is not based on the AI response length. It is based on the complexity and sentiment of the **incoming** customer message. The AI simulates "thinking about what was just said."

```text
incoming message is simple greeting only (Halo, Hi, Selamat pagi):
  T_base = 1.0s

incoming message is normal inquiry:
  T_base = 1.5s (default)

incoming message contains complex multi-part question:
  T_base = 2.0s

incoming message contains urgency signal (darurat, segera, emergency, gawat, buru):
  T_base = 1.0s  // read fast, respond faster

incoming message contains angry/frustrated signal:
  T_base = 2.5s  // take a beat; rushing feels dismissive
```

### Implementation in milliseconds

```text
WPM_char        = 250
T_max_ms        = 8000

T_base_ms = determine from incoming message sentiment (see above) × 1000
typing_ms = (response_length / WPM_char) × 60 × 1000
T_delay_ms = min(T_base_ms + typing_ms, T_max_ms)
```

## Layer 2 — Typing indicator (presence simulation)

Evolution API supports `sendPresence` with values `composing` and `paused`. Use this before every outbound message.

```text
1. AI decision ready, outbox row created
2. Outbox worker starts:
   a. call Evolution.sendPresence('composing', customer_jid)
   b. wait calculated_delay milliseconds
   c. call Evolution.sendText(content, customer_jid)
   d. call Evolution.sendPresence('paused', customer_jid)
   e. update outbox status = sent
```

Do not call `sendPresence` after message delivery. The paused state is set before move to next action so it does not overlap if sending multiple chunks.

## Layer 3 — Long message chunking

WhatsApp is a messaging app, not an email client. Long walls of text reduce engagement and feel robotic.

```text
chunk_threshold_chars = 280

if response_length <= chunk_threshold_chars:
  send as single message

if response_length > chunk_threshold_chars:
  split at natural paragraph breaks, not mid-sentence
  max 3 chunks per response
  for each chunk:
    send typing indicator
    wait inter_chunk_delay_ms (1200–1800ms)
    send chunk
    pause presence
```

Splitting rules:
- Split at double newline first
- If no double newline exists, split at sentence boundary near the midpoint
- Never split mid-word or mid-sentence
- Never produce a chunk shorter than 40 characters unless it is the final one

## Layer 4 — Read receipt awareness

Evolution emits read receipts when the customer opens the chat. Optionally use this to reduce delay for already-open conversations, since the customer is actively waiting.

```text
if customer_has_open_chat:
  calculated_delay *= 0.8
```

This is optional. Do not implement it before Layer 1 and Layer 2 are stable.

## Layer 5 — Message burst protection

Do not send multiple AI messages in rapid succession even when chunking. Rapid delivery of multiple messages looks like a spammer.

```text
minimum_inter_message_gap_ms = 1000

Always enforce this gap between consecutive outbound messages to the same customer,
regardless of whether they are chunks of one response or separate planned replies.
```

## BullMQ implementation pattern

Use a BullMQ delayed job for the pacing wait instead of `setTimeout`. This is safer under worker crashes and restarts.

```text
// When outbox row is created:
await pacingQueue.add(
  'send_message',
  { outbox_id: row.id, conversation_id: row.conversation_id },
  {
    delay: calculated_delay,
    jobId: `send:${row.id}`,   // idempotent, prevents duplicate sends
    attempts: 3,
    backoff: { type: 'exponential', delay: 2000 }
  }
)

// In the pacing worker:
// 1. Set typing indicator
// 2. Send via Evolution
// 3. Update outbox status
```

## Checklist

- [ ] Response delay is calculated from message length, not a constant
- [ ] Urgency override reduces delay
- [ ] Typing indicator is set before send, released after
- [ ] Long messages are chunked at paragraph boundaries
- [ ] Inter-chunk delay uses typing indicator
- [ ] Pacing is implemented as a delayed queue job, not setTimeout
- [ ] Minimum inter-message gap is enforced
- [ ] Humanization applies to all outbound messages, not only AI replies (system messages, notifications)
