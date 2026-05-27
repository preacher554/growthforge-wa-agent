# Production risks to call out

## Critical

- AI and human both reply to the same customer.
- Duplicate webhook causes duplicate AI replies or duplicate tool execution.
- Webhook blocks on model call and times out, causing provider retries.
- Same-number human replies are ignored because `fromMe` messages are dropped. This is deployment-blocking for same-number handoff.
- Retry sends a newly generated answer instead of the original persisted answer.
- Tenant data leaks through global memory, vector search, prompt context, or logs.
- Tool executes payment, discount, booking, invoice, refund, or customer mutation without approval.
- Invalid model string causes every LLM call to fail after deployment.

## High

- No per-conversation locking, allowing concurrent workers to respond.
- No message buffer, causing premature replies to fragmented WhatsApp messages.
- No outbox, making send status and retry behavior ambiguous.
- Handoff state can become stale forever.
- Model timeout fallback loses conversation continuity.
- Admin notification is sent but not audited.
- No dead-letter handling for failed webhooks or outbound sends.

## Medium

- Long replies are not chunked for WhatsApp.
- AI reply timing is unnaturally fast.
- Emotional state is detected only by prompt, not tracked as runtime signal.
- Business hours and SLA rules are hard-coded.
- No tenant-level budget, rate limit, or model fallback policy.
- Media, captions, voice notes, and buttons are unsupported without explicit fallback.
- No no-UI admin release mechanism for v1 handoff recovery.

## Design principle

Production WhatsApp AI is not primarily an intelligence benchmark. It is a distributed communication runtime. Prioritize ordering, ownership, memory continuity, idempotency, policy boundaries, and recovery.
