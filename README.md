# GrowthForge WA Agent

Managed AI WhatsApp Agent infrastructure for GrowthForge.

This repo is the source of truth for packaging, skills, prompts, onboarding, tenant configuration, and operating procedures for GrowthForge's WhatsApp AI product.

## Product focus v1

Build only the first two packages:

1. **Basic — AI Receptionist**
   - Answers FAQ and business info.
   - Captures simple customer needs.
   - Hands off to human admin when needed.

2. **Pro — AI Sales Receptionist**
   - Includes Basic.
   - Qualifies leads.
   - Handles light sales discovery and objections.
   - Produces lead summaries for human follow-up.

Advanced features such as SKU/catalog lookup, QRIS/VA payment, invoice generation, ongkir checks, Meta Ads integration, omnichannel, AI voice, and custom dashboards are intentionally out of v1 scope and belong to future add-ons or Enterprise scope.

## Operating model

GrowthForge starts as a **managed setup**, not a self-serve SaaS:

- Uses the customer's own WhatsApp business number.
- GrowthForge configures the agent, knowledge, persona, handoff rules, and monitoring.
- Each client is a separate tenant with isolated memory and data.
- Shared skills/templates live in this repo.

## Repo map

| Folder | Isi | Keterangan |
|---|---|---|
| `lia-profil/` | Persona + system prompt Lia | Profil identitas Lia (seperti "diri" Lia) |
| `skills/` | WA Agent Core, Basic, Pro | Skill operasional yang dipakai runtime |
| `runbook/` | SOP, arsitektur, integrasi | Dokumentasi operasional & teknis |
| `policies/` | Scope wall, data ownership | Aturan main & boundary |
| `tenants/examples/` | Contoh tenant Basic & Pro | Template konfigurasi per client |
| `onboarding/` | Intake form | Form pengumpulan data client |
| `templates/` | Lead summary template | Template ringkasan lead |
| `research/` | HaloAI reference | Riset kompetitor |

## Core principle

Do not sell "AI". Sell a business outcome:

- customer chats get answered faster,
- leads do not disappear,
- admins get cleaner handoff summaries,
- owners do not need to understand the technical stack.
