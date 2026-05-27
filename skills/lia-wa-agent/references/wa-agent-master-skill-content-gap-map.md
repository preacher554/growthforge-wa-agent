# WA Agent Master Skill — Content Gap Map from External Template Libraries

Use this reference when reviewing external customer-service/template repos to improve the GrowthForge WA Agent master skill.

## Core lesson

External template repos are usually strongest at the **content library layer** (scenario coverage, templated replies, variable placeholders, platform/industry taxonomies). The GrowthForge WA Agent master skill is strongest at the **operating system layer** (architecture, packages, scope wall, tenant isolation, handoff rules, runtime integration).

The usual gap is **not** architecture. The usual gap is a missing or thin **shared playbook/content layer** for reusable replies and scenario handling.

## What to add first

Highest-value additions to the master skill:

1. Shared reply pattern library
   - greetings
   - pricing boundary replies
   - missing-info safe replies
   - handoff transition language
   - post-handoff waiting language
   - complaint de-escalation
   - light follow-up / re-engagement

2. Scenario taxonomy
   - presales
   - sales
   - aftersales
   - handoff
   - objection handling
   - follow-up

3. Approved objection-response library
   - "too expensive"
   - "I'll think about it"
   - "what's the difference"
   - "can you do custom"
   - "is this suitable for my business"

4. Aftersales safety pack
   - apology language
   - fact collection prompts
   - escalation thresholds
   - no-overpromise resolution language

5. Reply QA rubric
   - too long / too many questions
   - invented claims
   - premature handoff
   - robotic tone
   - weak next-step question

6. Variable placeholder contract
   - prefer stable placeholders like `{{customer_name}}`, `{{business_name}}`, `{{service_name}}`, `{{approved_price}}`, `{{handoff_reason}}`

7. Vertical starter packs
   - beauty services
   - local services
   - courses / education
   - B2B agency

## Recommended repo shape

```txt
playbooks/
  general/
    presales.md
    sales.md
    aftersales.md
    handoff.md
  objections/
    common-objections.md
    custom-scope-boundary.md
  followup/
    light-followup.md
    re-engagement.md

verticals/
  beauty-services.md
  local-services.md
  courses-education.md
  b2b-agency.md

qa/
  reply-rubric.md
  conversation-review-checklist.md

templates/
  template-variable-contract.md
```

## What NOT to import blindly

Do **not** dump large marketplace-specific libraries directly into the shared master skill unless the product scope explicitly includes them.

Avoid bulk inclusion of:
- Amazon/Shopee/Lazada/eBay/Shopify policy-heavy scripts
- refund/return/exchange flows that depend on tenant policy
- SKU/order/payment logic outside approved package scope

Reason:
- increases prompt noise
- weakens scope wall
- mixes ecommerce-specific assumptions into general receptionist/sales behavior
- creates policy risk if replies imply capabilities the tenant does not actually support

## Safe adoption rule

When borrowing from a reference repo:
1. Extract the **taxonomy and response pattern**, not the tenant/platform policy text.
2. Convert examples into GrowthForge-safe, tenant-agnostic playbooks.
3. Keep package boundaries explicit (Basic/Pro vs add-on/custom).
4. Prefer concise Indonesian customer-facing patterns for Lia-style deployments unless the tenant requires another language.
5. If a scenario depends on order IDs, stock, refunds, shipping carriers, or payment state, treat it as add-on/custom or tenant-specific until proven otherwise.

## Review question set

When inspecting an external template repo, ask:
- Does this improve shared scenario coverage?
- Is it tenant-agnostic?
- Does it fit Basic, Pro, or only an add-on?
- Can the lesson be represented as a playbook instead of a hardcoded script?
- Will this reduce prompt drift or increase it?
- Does it help QA/auditability?
