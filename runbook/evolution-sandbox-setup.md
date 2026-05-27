# Evolution Sandbox Setup — Lia / GrowthForge

## Purpose

Local/private Evolution API gateway for Lia, GrowthForge's internal WA Agent.

## Runtime location

```txt
/root/services/evolution-growthforge
```

## Services

- `gf-evolution-api` — Evolution API v2 gateway.
- `gf-evolution-manager` — Manager UI.
- `gf-evolution-postgres` — internal Evolution database.
- `gf-evolution-redis` — internal cache/session support.

## Ports

Bound to localhost only:

```txt
API:        http://127.0.0.1:8080
Manager UI: http://127.0.0.1:3002
```

Ports `3000` and `3001` were already occupied by existing Node/Vite processes, so Manager uses `3002`.

## Instance

```txt
lia-growthforge
```

Integration:

```txt
WHATSAPP-BAILEYS
```

Current intended use:

```txt
GrowthForge internal WhatsApp frontdesk / Lia sandbox
```

## Secrets

Secrets are stored only in:

```txt
/root/services/evolution-growthforge/.env
```

Do not commit or print:

- `AUTHENTICATION_API_KEY`
- `POSTGRES_PASSWORD`
- `REDIS_PASSWORD`
- instance tokens

## Manager image note

`evoapicloud/evolution-manager:latest` had an nginx config issue with `gzip_proxied ... must-revalidate ...` on this host/image combination.

A safe local override was added:

```txt
/root/services/evolution-growthforge/manager-nginx.conf
```

## Model routing decision

Lia runtime should use:

```txt
openai_oauth
```

Not OpenRouter for Lia's primary brain, per Chief's decision.

## Next steps

1. Chief sets up Supabase.
2. Create `whatsapp-agent-architect-runtime` backend.
3. Runtime receives Evolution webhooks.
4. Runtime loads WA Agent skills and Lia persona.
5. Runtime uses OpenAI OAuth model routing.
6. Runtime implements same-number human handoff:
   - notify admin private WhatsApp,
   - pause AI,
   - let admin reply through the business/agent WhatsApp number.
