# Lia Runtime Debugging Notes

Use this reference when Lia receives WhatsApp messages but customers get no reply.

## Silent-reply symptom checklist

1. Confirm runtime is listening and healthy:
   ```bash
   ss -ltnp | grep ':3300'
   curl -sS http://127.0.0.1:3300/health
   ```
2. Confirm Evolution webhook points to Lia runtime and is enabled:
   ```bash
   set -a; . /root/services/evolution-growthforge/.env; set +a
   curl -sS -H "apikey: ${AUTHENTICATION_API_KEY}" \
     http://127.0.0.1:8080/webhook/find/lia-growthforge | python -m json.tool
   ```
3. Check DB shape, not only process health:
   ```bash
   cd /root/repos/whatsapp-agent-architect-runtime
   python - <<'PY'
   from app.config import load_settings
   import psycopg
   s=load_settings()
   with psycopg.connect(s.database_url) as conn:
       with conn.cursor() as cur:
           cur.execute('select count(*) from conversations')
           print('conversations', cur.fetchone()[0])
           cur.execute('select count(*) from messages')
           print('messages', cur.fetchone()[0])
           cur.execute('select direction, left(text,80), created_at from messages order by created_at desc limit 8')
           for row in cur.fetchall(): print(row)
   PY
   ```
   If inbound exists but outbound does not, inspect webhook logic around dedupe/sendText.
4. For duplicate handling, verify order in `app/main.py`:
   - duplicate SELECT before inbound insert
   - history fetched before inbound insert
   - inbound insert before generate/send/outbound insert
5. Verify model route independently before blaming Evolution:
   ```bash
   hermes chat -Q --provider openai-codex -m gpt-5.2 \
     -q 'Reply with exactly LIA_MODEL_OK and nothing else.'
   ```
6. Run regression tests:
   ```bash
   python -m pytest -q
   ```

## Model switch checklist

- Runtime repo: `/root/repos/whatsapp-agent-architect-runtime`
- Edit `app/config.py`: `hermes_model_provider`, `hermes_model`
- Restart service/process on port 3300
- Verify settings by importing `load_settings()` from runtime repo
- Dashboard repo: `/root/repos/Growth-Forge-WEB-UI-dashboard`
- Update `src/app/api/wa-agents/route.ts` fallback model label when needed
- Run `npm run build`
- Restart dashboard PM2 process: `pm2 restart growthforge-mission-monitor --update-env`
- Verify the live port, not just source/build output:
  ```bash
  curl -sS http://127.0.0.1:3200/api/wa-agents | python -m json.tool | grep '"model"'
  ```
- If the browser still shows the previous model after source/build is correct, suspect a stale Next/PM2 bundle first; restart PM2 and hard-refresh before changing runtime code again.

## Known root cause captured May 2026

Bug: inbound message was inserted before dedupe check. The duplicate SELECT immediately found the just-inserted row, so every new message returned `duplicate_ignored`. Health stayed green, DB showed inbound rows, but no outbound rows were created.

Fix: move duplicate check before insert, fetch history before inserting current inbound, and add webhook regression tests.
