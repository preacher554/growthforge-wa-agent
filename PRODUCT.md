# WA Agent — Product Definition

## One-line positioning

WA Agent is a managed AI WhatsApp receptionist and sales assistant for businesses, using the client's own WhatsApp number.

## What we sell

We do **not** sell raw AI, LLMs, prompts, vector databases, or automation tooling.

We sell a configured AI employee capability:

- **Basic:** AI Receptionist.
- **Pro:** AI Sales Receptionist.
- **Custom/Add-on:** Custom workflow, payment, catalog, integrations.

## Delivery model

This is initially a managed service:

1. Client provides WhatsApp number and business information.
2. Provider configures the AI agent.
3. Agent answers inbound WhatsApp messages based on approved knowledge and scope.
4. Agent escalates to human admin when needed.
5. Provider monitors, optimizes, and maintains the setup.

## Strategic positioning

Competitive platforms position as all-in-one AI end user experience SaaS with AI Chat, AI Voice, AI CRM, payments, ads integration, and enterprise scale.

Our v1 wedge is narrower:

- Focus on **WhatsApp-native interaction**.
- **Human-AI collaboration** (not AI-only).
- **Same-number handoff** — admin and AI share one business number.
- **Basic and Pro tiers only** — no feature bloat.

## Value proposition

For the client (business owner):
- Never miss a WhatsApp lead — AI answers 24/7.
- Leads are qualified before reaching human admin.
- No extra app or tool to learn — runs on their existing WhatsApp Business number.

For the provider (GrowthForge):
- Recurring monthly revenue per managed agent.
- Upsell path: Basic → Pro → Custom add-ons.
- Operational leverage: one runtime codebase serves multiple clients via multi-tenancy.

## Terminology

| Term | Meaning |
|---|---|
| **Agent** | The AI receptionist/sales assistant instance. Named by the client (e.g., "Mona", "Rara", "Bara"). |
| **Client** | The business that pays for the managed WA Agent service. |
| **End user / end user** | The client's own end user who messages the business WhatsApp number. |
| **Tenant** | An isolated agent instance (one client = one tenant). |
| **Admin** | The human admin on the client's side who can take over conversations. |
| **Provider** | GrowthForge, the managed service provider. |

## Pricing philosophy

- **Per-agent monthly subscription** (not per-message, not per-seat).
- Basic and Pro have fixed pricing published on the website.
- Custom/Add-on is scoped and quoted per engagement.
- Setup/onboarding fee may apply for Custom tier.

## Scalability notes

- Phase 1: One managed runtime per client (simplest ops).
- Phase 2: Shared runtime with tenant isolation (better unit economics).
- Phase 3: Self-serve onboarding with template-based configuration.
