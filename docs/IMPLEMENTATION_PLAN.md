# Implementation Plan — Basic + Pro Infrastructure

## Goal

Create a clean source-of-truth repo for GrowthForge WA Agent Basic and Pro packages.

## Phase 1 — Documentation foundation

- [x] README.
- [x] Product definition.
- [x] Package structure.
- [x] HaloAI reference notes.

## Phase 2 — Operating skills

- [x] Core rules.
- [x] Basic package skill.
- [x] Pro package skill.
- [x] Lia persona.

## Phase 3 — Prompt and onboarding

- [x] Lia system prompt.
- [x] Client intake form.
- [x] Human handoff SOP.
- [x] Scope wall policy.

## Phase 4 — Tenant examples

- [x] Basic service tenant example.
- [x] Pro service tenant example.

## Next build tasks

1. Add JSON/YAML schema validation for `tenant.yaml`.
2. Add a simple prompt compiler that combines:
   - core skill,
   - package skill,
   - tenant profile,
   - FAQ,
   - handoff rules.
3. Add a CLI dry-run command:
   - input tenant ID + sample message,
   - output compiled prompt context and expected routing.
4. Add minimal tests for tenant validation and package capability boundaries.
5. Add Evolution API integration notes after the product docs are stable.
