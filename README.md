# GrowthForge WA Agent — Master Skill

**Shared knowledge base** untuk semua WA Agent GrowthForge. Repo ini berisi skill, policy, arsitektur, dan referensi — **bukan** data tenant atau persona spesifik.

> **Persona per agent ada di repo sendiri**, cek folder `WA_Agent_Persona_*/`.

## Repo map

| Folder | Isi | Keterangan |
|---|---|---|
| `skills/` | WA Agent Core, Basic, Pro | Skill operasional (dipakai semua agent) |
| `runbook/` | SOP, arsitektur, integrasi | Dokumentasi operasional & teknis |
| `policies/` | Scope wall, data ownership | Aturan main & boundary |
| `onboarding/` | Intake form | Form pengumpulan data client |
| `templates/` | Lead summary template | Template ringkasan lead |
| `research/` | HaloAI reference | Riset kompetitor |
| `PRODUCT.md` | Definisi produk | Positioning & paket |
| `PACKAGE_STRUCTURE.md` | Detail Basic & Pro | Capability masing-masing paket |

## Arsitektur repo

```
growthforge-wa-agent/            ← MASTER SKILL (repo ini)
  skills/                        ← Shared capabilities
  runbook/                       ← Shared SOP & docs
  policies/                      ← Shared rules
  ...

WA_Agent_Persona_Lia/           ← PER-AGENT (repo terpisah)
  persona.md                     ← Identitas agent
  system-prompt.md               ← System prompt agent
  business-profile.md            ← Info bisnis client
  faq.md                         ← FAQ client
  handoff-rules.md               ← Aturan handoff
  tenant.yaml                    ← Metadata agent

WA_Agent_Persona_ClientA/       ← Agent lain (repo terpisah)
  ...
```

## Cara spawn agent baru

1. Buat repo baru: `WA_Agent_Persona_<NamaAgent>`
2. Copy template dari `WA_Agent_Persona_Lia/` sebagai starting point
3. Isi `persona.md`, `business-profile.md`, `faq.md`, `handoff-rules.md`, `tenant.yaml`
4. Hubungkan ke runtime via `tenant.yaml`

## Core principle

Do not sell "AI". Sell a business outcome:

- customer chats get answered faster,
- leads do not disappear,
- admins get cleaner handoff summaries,
- owners do not need to understand the technical stack.
