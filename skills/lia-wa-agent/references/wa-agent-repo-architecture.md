# WA Agent Repo Architecture (v2 — post May 2026 restructure)

## 3-Layer Repo Model

```
growthforge-wa-agent/              ← MASTER SKILL (shared, public)
  skills/                          ← Core, Basic, Pro operating skills
  runbook/                         ← SOP, architecture, integration docs
  policies/                        ← Scope wall, data ownership
  onboarding/                      ← Client intake form
  templates/                       ← Lead summary template
  research/                        ← Competitor analysis
  PRODUCT.md + PACKAGE_STRUCTURE.md

WA_Agent_Persona_<AgentName>/      ← PER-AGENT (repo per customer)
  persona.md                       ← Identity + personality
  system-prompt.md                 ← Agent system prompt
  business-profile.md              ← Client business info
  faq.md                           ← Client FAQ
  handoff-rules.md                 ← Escalation triggers + targets
  tenant.yaml                      ← Metadata: capabilities, contacts, memory
  README.md                        ← Agent info + skill dependencies

growthforge-wa-agent-runtime/      ← RUNTIME (code, private)
  app/                             ← FastAPI webhook handler
  src/                             ← Dashboard frontend
```

## Key Principle

- **Master skill** = shared knowledge, never tenant-specific data
- **Persona repo** = per-agent config, isolated per customer
- **Runtime** = engine that loads persona + skills at runtime

## Spawn New Agent Workflow

1. Create new GitHub repo: `WA_Agent_Persona_<AgentName>`
2. Copy template from `WA_Agent_Persona_Lia/` as starting point
3. Fill in: persona.md, business-profile.md, faq.md, handoff-rules.md, tenant.yaml
4. Connect to runtime via tenant.yaml → whatsapp_instance mapping
5. Runtime loads skills from growthforge-wa-agent (master) + persona from agent repo

## GitHub Repo Creation

```bash
# Via GitHub API (requires valid GITHUB_TOKEN with repo scope)
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/user/repos \
  -d '{"name":"WA_Agent_Persona_<Name>","description":"...","private":false}'
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
