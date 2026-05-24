# GrowthForge WA Agent Product Definition

## One-line positioning

GrowthForge WA Agent is a managed AI WhatsApp receptionist and sales assistant for Indonesian SMBs, using the customer's own WhatsApp number.

## What we sell

We do **not** sell raw AI, LLMs, prompts, vector databases, or automation tooling.

We sell a configured AI employee capability:

- **Basic:** AI Receptionist.
- **Pro:** AI Sales Receptionist.

## Delivery model

This is initially a managed service:

1. Customer provides WhatsApp number and business information.
2. GrowthForge configures the AI agent.
3. Agent answers inbound WhatsApp messages based on approved knowledge and scope.
4. Agent escalates to human admin when needed.
5. GrowthForge monitors, optimizes, and maintains the setup monthly.

## Strategic wedge

HaloAI-style platforms position as all-in-one AI customer experience SaaS with AI Chat, AI Voice, AI CRM, payments, ads integration, and enterprise scale.

GrowthForge's v1 wedge is narrower:

> AI admin WhatsApp yang praktis, paketnya jelas, setup dibantu, dan cocok untuk UMKM.

## Not v1

These are intentionally excluded from Basic/Pro defaults:

- SKU/catalog lookup.
- Stock checks.
- QRIS/VA payment verification.
- Invoice generation.
- Ongkir automation.
- Marketplace integration.
- Meta Ads conversion API.
- Omnichannel inbox.
- AI voice/call center.
- Custom CRM/ERP integrations.
- Custom dashboards.

They can become add-ons or Enterprise scope later.

## Tenant model

Each business/customer is a tenant.

Every tenant has isolated:

- WhatsApp instance/number,
- package selection,
- business profile,
- FAQ/knowledge base,
- customer memory namespace,
- handoff contacts,
- logs and analytics,
- billing and usage status.

Shared skills and operating rules come from this repo.

## Success criteria v1

Basic works when:

- common questions are answered accurately,
- AI never invents unavailable information,
- handoff is clean and timely,
- customer feels heard,
- admin load decreases.

Pro works when:

- AI asks useful qualification questions,
- lead intent is captured,
- hot/warm/cold status is reasonable,
- human admin receives a concise summary,
- AI avoids overpromising price, discounts, guarantees, or custom terms.
