# Runtime module and folder architecture

## Recommended folder structure

```text
growthforge-wa-agent-runtime/
├── src/
│   │
│   ├── gateway/                        # Webhook entry point — must return 200 fast
│   │   ├── webhook.handler.ts          # Parse, validate, idempotency check, enqueue
│   │   ├── idempotency.ts              # INSERT webhook_events with UNIQUE conflict check
│   │   ├── payload.normalizer.ts       # Evolution raw payload → internal event shape
│   │   └── frommie.classifier.ts       # fromMe=true → AI/system vs human_admin
│   │
│   ├── queue/                          # BullMQ queue definitions and workers
│   │   ├── queues.ts                   # All named queues: buffer, planning, outbox, pacing
│   │   └── workers/
│   │       ├── message.worker.ts       # Buffer open/reset/flush coordination
│   │       ├── planning.worker.ts      # AI context build, LLM call, signal extraction
│   │       ├── pacing.worker.ts        # Typing indicator, delay, send trigger
│   │       └── outbox.worker.ts        # Evolution send, retry, dead-letter
│   │
│   ├── runtime/                        # Core conversation runtime logic
│   │   ├── state-machine.ts            # FSM transitions with audit emit
│   │   ├── ownership.engine.ts         # AI/human/system lock acquire and release
│   │   ├── buffer.manager.ts           # Fragment accumulation and merge
│   │   ├── humanizer.ts                # Delay calculation and typing indicator
│   │   ├── chunker.ts                  # Long message splitting
│   │   └── continuity.summarizer.ts    # human_active session → AI context summary
│   │
│   ├── ai/                             # AI planning layer
│   │   ├── planner.ts                  # Orchestrate context build → LLM → signals
│   │   ├── context.builder.ts          # Assemble layers: system/tenant/memory/history/turn
│   │   ├── signal.extractor.ts         # Parse structured signal block from LLM output
│   │   └── model.router.ts             # Per-tenant model selection and fallback
│   │
│   ├── memory/                         # Memory read and write
│   │   ├── conversation.memory.ts      # Recent message history fetch
│   │   ├── customer.memory.ts          # Cross-session profile, lead, history
│   │   └── summary.writer.ts           # Write session_end, human_session, handoff summaries
│   │
│   ├── tenants/                        # Tenant config and capability
│   │   ├── tenant.loader.ts            # Load tenant row + config from DB
│   │   ├── prompt.compiler.ts          # Assemble system prompt from tenant data
│   │   └── capability.checker.ts       # Is this action allowed for this tenant/package?
│   │
│   ├── outbox/                         # Safe send pattern
│   │   ├── outbox.writer.ts            # Create outbox row before sending
│   │   └── outbox.sender.ts            # Call Evolution, update status, handle failure
│   │
│   ├── handoff/                        # Human-AI collaboration
│   │   ├── handoff.trigger.ts          # Detect trigger, write session, notify admin
│   │   ├── admin.notifier.ts           # Send notification to admin private JID
│   │   ├── release.handler.ts          # Process RESUME command or timeout
│   │   └── stale.recovery.ts           # Auto-release conversations stuck in handoff
│   │
│   ├── observability/                  # Event log and diagnostics
│   │   ├── event.emitter.ts            # emit(event_type, payload) → runtime_events
│   │   ├── metrics.ts                  # Counters and latency tracking
│   │   └── dead.letter.ts             # Write dead_letters on exhausted retries
│   │
│   └── db/                             # Database access
│       ├── client.ts                   # Supabase + transaction pooler setup
│       ├── schema.sql                  # Full schema (see database-schema reference)
│       └── migrations/
│
├── test/
│   └── scenarios/
│       ├── fragment-burst.test.ts      # Buffer timer reset, single merged reply
│       ├── human-takeover.test.ts      # fromMe → human_active, AI silent
│       ├── duplicate-webhook.test.ts   # Same message_id arrives twice, deduped
│       ├── outbox-retry.test.ts        # Send fails, retry reads same content
│       ├── continuity-resume.test.ts   # Human session summary injected on AI resume
│       └── stale-lock-recovery.test.ts # Conversation stuck in ai_planning auto-recovered
│
└── infra/
    ├── docker-compose.yml              # Redis + local Evolution for dev
    └── lia-runtime.service             # systemd unit
```

## Module responsibilities

### gateway/

The webhook handler must do nothing except:
1. Parse and validate the incoming payload
2. Run the idempotency check (INSERT with UNIQUE conflict)
3. Classify fromMe events (outbox match vs human admin)
4. Enqueue the normalized event to the message queue
5. Return HTTP 200

It must NOT call the LLM, call Evolution, read conversation state, or do database writes beyond the idempotency log.

### queue/workers/

Each worker owns one phase of the message lifecycle:

| Worker | Input | Output |
|---|---|---|
| message.worker | webhook event enqueued | buffer opened/appended or immediate flush |
| planning.worker | buffer.flushed event | outbox row created |
| pacing.worker | outbox.created event | Evolution.sendText called |
| outbox.worker | send result | outbox status updated, retry or dead-letter |

Workers must be idempotent. A job that runs twice must produce the same result as running once.

### runtime/state-machine.ts

Every state transition must:
1. Verify the transition is valid from the current state
2. Update `conversations.state` and `conversations.state_updated_at`
3. Emit `state.transitioned` to runtime_events

No module may update `conversations.state` directly. All updates go through the state machine.

### ai/planner.ts

The planner orchestrates but does not implement:
1. Call `context.builder.ts` to assemble the full context
2. Call the LLM
3. Call `signal.extractor.ts` on the response
4. Write the AI decision to `ai_decisions` table
5. Write signals to `conversation_signals`
6. Write outbox row via `outbox.writer.ts`

The planner never calls `Evolution.sendText` directly.

## Queue topology

```text
Queue: message-events      FIFO per conversation_id
Queue: buffer-flush        Delayed jobs, jobId = 'buffer_flush:{conversation_id}'
Queue: planning            FIFO per conversation_id
Queue: outbox-send         FIFO per conversation_id, delayed by pacing
Queue: handoff-notify      Standard, low volume
Queue: stale-recovery      Scheduled, runs every 5 minutes

Dead-letter handling: BullMQ built-in failed queue, reviewed via admin command or future UI
```

## Queue topology

```text
Queue: message-events      FIFO per conversation_id  (inbound webhook processing)
Queue: buffer-flush        Delayed jobs, jobId = 'buf:{conversation_id}'
Queue: planning            FIFO per conversation_id  (LLM call + signal extraction)
Queue: outbox-send         FIFO per conversation_id, delayed by pacing calculation
Queue: handoff-notify      Standard queue, low volume
Queue: stale-recovery      Scheduled, runs every 5 minutes
Queue: circuit-check       Scheduled, monitors open circuit breakers

Dead-letter: BullMQ built-in failed queue → mirrored to dead_letters table
```

## Technology dependencies

| Component | Technology | v1 Recommendation | Scale Alternative |
|---|---|---|---|
| WhatsApp bridge | Evolution API v2 (Baileys) | ✅ In use | Official Meta Cloud API |
| Persistence | Supabase (Postgres, transaction pooler) | ✅ In use | Self-hosted Postgres + PgBouncer |
| Queue and delayed jobs | BullMQ + Redis | ✅ Default for Node.js | Apache Kafka (see below) |
| Session state cache | Redis | Add with BullMQ | Redis Cluster |
| LLM | OpenAI direct API | ✅ In use | OpenRouter for multi-model fallback |
| Runtime | Node.js (TypeScript) | ✅ In use | — |
| Infra | VPS + systemd | ✅ In use | Docker + managed k8s at scale |

## Kafka topology (scale-up path)

When active tenant count exceeds ~50 simultaneous conversations or throughput exceeds BullMQ's reliable operating range, migrate the inbound pipeline to Kafka:

```text
Topic: wa-raw-events
  Partition key: null (round-robin; Ingress Gateway is stateless)
  Purpose: holds raw webhook payloads directly from Evolution

Topic: wa-session-events
  Partition key: phone_number (CRITICAL: ensures FIFO per customer across all workers)
  Purpose: holds aggregated, deduplicated, normalized events ready for FSM processing

Consumer group: session-actor-workers
  - Each worker processes one partition
  - FIFO per customer guaranteed by partition key
  - No per-conversation BullMQ lock needed at this scale
```

Why Kafka provides stronger FIFO guarantee than BullMQ at scale:
- Kafka partition = physical ordering guarantee enforced by the broker
- BullMQ FIFO = application-level ordering that can be violated under worker contention
- For SMB v1 with < 50 concurrent conversations: BullMQ is correct and simpler
- For 500+ concurrent conversations: Kafka partition-by-phone is the right architecture

Do not migrate to Kafka before BullMQ is stable. Kafka adds significant operational complexity.

Do not add infrastructure complexity before the Phase 0 and Phase 1 correctness work is complete.
