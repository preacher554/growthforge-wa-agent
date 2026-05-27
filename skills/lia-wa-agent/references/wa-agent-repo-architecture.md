# WA Agent Repo Architecture (v2.1 — May 2026 restructure)

## Repo & Folder Structure

```txt
/root/repos/
├── whatsapp-agent-architect/           ← MASTER SKILL (shared, GitHub preacher554)
│   ├── skills/                     ← Core, Basic, Pro + lia-wa-agent skill
│   ├── runbook/                    ← SOP, architecture, integration docs
│   ├── policies/                   ← Scope wall, data ownership
│   ├── onboarding/                 ← Client intake form
│   ├── templates/                  ← Lead summary template
│   ├── research/                   ← Competitor analysis
│   └── PRODUCT.md + PACKAGE_STRUCTURE.md
│
├── WA_Agent_Persona_Lia/           ← LIA INTERNAL (GitHub preacher554)
│   ├── persona.md                  ← Identity + personality
│   ├── system-prompt.md            ← Agent system prompt
│   ├── business-profile.md         ← Business info
│   ├── faq.md                      ← FAQ
│   ├── handoff-rules.md            ← Escalation triggers
│   └── tenant.yaml                 ← Config (tenant_id, model, instance)
│
├── whatsapp-agent-architect-runtime/   ← RUNTIME (private, no GitHub)
│   ├── app/                        ← FastAPI webhook handler
│   └── src/                        ← Dashboard frontend
│
└── wa-customers/                   ← KONSUMEN (Nara-managed)
    ├── <customer-slug-A>/          ← Per-customer folder
    │   ├── persona.md
    │   ├── system-prompt.md
    │   ├── business-profile.md
    │   ├── faq.md
    │   ├── handoff-rules.md
    │   └── tenant.yaml
    └── <customer-slug-B>/
        └── ...
```

## Key Principles

- **Master skill** = shared knowledge only, never tenant-specific data
- **Lia persona** = internal GrowthForge, GitHub repo for version control
- **Customer persona** = per-customer folder under `wa-customers/`, managed by Nara
- **Runtime** = engine that loads persona config at runtime via `tenant_id`
- Each customer can have different persona + package (Basic/Custom)

## Spawn New Customer Agent Workflow

1. Create folder: `/root/repos/wa-customers/<customer-slug>/`
2. Copy template from `WA_Agent_Persona_Lia/` as starting point
3. Fill in: persona.md, system-prompt.md, business-profile.md, faq.md, handoff-rules.md, tenant.yaml
4. Set unique `tenant_id` and `whatsapp_instance` in tenant.yaml
5. Optional: if customer needs GitHub repo, create `WA_Agent_Persona_<Name>` via GitHub API and push
6. Runtime auto-loads config via `tenant_id` + `whatsapp_instance` mapping

## GitHub Repo Creation (Optional for Customers)

```bash
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/user/repos \
  -d '{"name":"WA_Agent_Persona_<Name>","description":"WA Agent persona for <Customer>","private":false}'
```

## Multi-Platform Profile Setup

Each Nara profile reads platform credentials from its `.env`:

```
# ~/.hermes/profiles/growthforge-wa-operator/.env
TELEGRAM_BOT_TOKEN=xxx
DISCORD_BOT_TOKEN=xxx
DISCORD_HOME_CHANNEL=1508741985266045029
DISCORD_REQUIRE_MENTION=true
```

Each bot platform needs its own unique token. Two profiles cannot share the same bot token.
