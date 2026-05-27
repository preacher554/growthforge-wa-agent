# Lia Same-Number Human Takeover Implementation Notes

Session-derived implementation details for the user-approved WA Agent behavior: when Chief/admin replies manually from the same WhatsApp Business number, Lia pauses for a 1-hour human window and resumes only on the next customer inbound after the window.

## Trigger and state model

- `waiting_human`: Lia has escalated and is waiting for human takeover.
- `human_active`: a human/admin has replied from the same WA Business number; Lia should stay silent while the human window is active.
- `ai_active`: Lia can auto-reply.

Evolution sent-message events from the WA Business app normally arrive with:

```json
{
  "data": {
    "key": {
      "remoteJid": "628xxx@s.whatsapp.net",
      "fromMe": true,
      "id": "..."
    },
    "message": {"conversation": "..."}
  }
}
```

## Runtime pattern

In the webhook handler, do not reject `fromMe:true` before conversation lookup. Instead:

1. Extract the message and ensure it has text + a private WhatsApp JID.
2. Resolve tenant and conversation.
3. Run dedupe before insert.
4. If `incoming.from_me`:
   - persist it as an `outbound` message with raw payload;
   - if current state is `waiting_human` or `human_active`, set state to `human_active`;
   - return without generating/sending an AI reply.
5. If inbound customer message while `human_active`:
   - check latest human outbound timestamp;
   - if less than 1 hour ago, store the inbound and return paused;
   - if 1 hour or more, set `ai_active`, generate reply, and include a resume note in the prompt.

## Prompt/history nuance

When building history for model context, distinguish human outbound messages from Lia outbound messages. If a stored outbound row has `raw.key.fromMe == true`, label it as `Admin GrowthForge`, not `Lia`. This prevents Lia from treating Chief's manual deal/closing messages as her own prior replies.

On resume, pass an operational note like:

```txt
Percakapan ini baru di-resume otomatis setelah admin/human GrowthForge mengambil alih. Balas sebagai Lia/WA Agent yang aktif kembali. Jangan ulang dari awal; lanjutkan natural dari konteks chat. Jika cocok, awali singkat dengan 'Aku Lia bantu lanjut ya Kak.'
```

## Verification tests to keep

Add/keep webhook tests for:

- `fromMe:true` sent-message while `waiting_human` → inserts outbound, sets `human_active`, no AI reply.
- Customer inbound during the 1-hour `human_active` window → inserted/stored, returns paused, no sendText.
- Customer inbound after 1 hour from the last human outbound → sets `ai_active`, calls `generate_reply` with resume context, sends reply.
- Existing dedupe and handoff notification tests still pass.

## Pitfalls

- Do not make the timer proactively send a WhatsApp message. Timer expiry only makes the next customer inbound eligible for AI response.
- Do not use customer `/resume` as the only resume path; same-number human takeover needs `fromMe:true` handling.
- Do not globally classify all outbound messages as Lia in history; manual human replies are operational context.
- New human/admin messages should reset the effective 1-hour window because the latest human outbound timestamp is used.
