# alfadocs/lovable

Lovable **skills** and **workspace knowledge** that keep AlfaDocs builders inside
the guardrails. This repo is the **source of truth** — edit here, then import into
Lovable.

## What's here

```
workspace-knowledge.md   → paste into Lovable Workspace Knowledge (always in context)
skills/                  → import into Lovable as workspace skills (loaded on demand)

  Build skills — how to build it right
    alfadocs-connected-app-oauth/     "Sign in with AlfaDocs" (OAuth2 + PKCE BFF)
    alfadocs-connected-app-api-key/   single-practice tool with an API key
    alfadocs-staff-sso-app/           internal staff tool (Google Workspace SSO + own data)
    alfadocs-google-workspace-auth/   @alfadocs.com staff login via Supabase Auth
    alfadocs-prototype-app/           internal mockups & prototypes (no backend)
    alfadocs-design-system/           build UI with @alfadocs/ui-kit
    alfadocs-webhooks/                receive & parse AlfaDocs webhooks
    alfadocs-oauth-token-refresh/     concurrency-safe token refresh
    alfadocs-gdpr-consent/            explicit consent for patient data
    alfadocs-patient-deep-link/       deep-link back into AlfaDocs
    alfadocs-typescript-setup/        the strict tsconfig.json baseline (Standard §3)

  Review skills — catch problems before shipping
    alfadocs-sanity-check/            fast "does it run & connect?" pre-ship gate (run first)
    alfadocs-code-review/             correctness, reuse/DRY, readability, errors
    alfadocs-security-review/         secrets, data isolation, tokens, XSS/CSP
    alfadocs-design-review/           design-system compliance
    alfadocs-logic-review/            practiceId scoping, webhooks, refresh logic
    alfadocs-architecture-review/     BFF shape, RLS, multi-tenancy
    alfadocs-accessibility-review/    keyboard, focus, contrast, RTL, labels
```

Each skill is a folder with a `SKILL.md` (+ optional supporting `.md` files). **Build**
skills fire while you're creating a feature; **review** skills fire when you ask Lovable
to review/audit an app (or before shipping).

**Knowledge vs skills.** Workspace knowledge is *always* loaded — it holds the
**Engineering Standard**: modularity & layering, type safety, tenant isolation,
security, webhooks, and the design system, with a skills index at the end. Skills
are *on-demand* playbooks — Lovable loads one only when its `description` matches
the task at hand. The review skills cite the Standard's section numbers (e.g. §12)
so a review enforces it 1:1.

## How to use it (in the `@alfadocs_builders` workspace)

1. **Workspace knowledge** — open **Settings → Knowledge**, paste the contents of
   [`workspace-knowledge.md`](./workspace-knowledge.md) into Workspace Knowledge.
   (Owners/admins only; ≤10,000 characters.)
2. **Skills** — Lovable imports **one skill per import**, and the source must have
   `SKILL.md` at its root. The repo-root URL therefore does **not** work (there's no
   root `SKILL.md` — just `skills/`). Instead, in **Skills → Add → Import from
   GitHub**, paste each skill's **subfolder URL** (the folder that contains its
   `SKILL.md`) — one import per skill:

   _Build_
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-connected-app-oauth`
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-connected-app-api-key`
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-staff-sso-app`
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-google-workspace-auth`
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-prototype-app`
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-design-system`
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-webhooks`
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-oauth-token-refresh`
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-gdpr-consent`
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-patient-deep-link`
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-typescript-setup`

   _Review_
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-sanity-check`
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-code-review`
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-security-review`
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-design-review`
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-logic-review`
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-architecture-review`
   - `https://github.com/alfadocs/ai-harness-instructions/tree/main/lovable/skills/alfadocs-accessibility-review`

   Each then loads automatically when its `description` matches the task. (A direct
   `…/blob/main/skills/<name>/SKILL.md` file URL works too — Lovable imports the
   parent folder.)

## Editing

Each skill is a folder with a `SKILL.md`:

```
---
name: <lowercase-hyphenated>
description: Use when … (the trigger Lovable matches on)
---
# … instructions …
```

Keep facts current: AlfaDocs apps are built in **Lovable + Supabase**; secrets live
in **Supabase Edge Function secrets**; `@alfadocs/ui-kit` is on **public npm**; the
design system catalogue is at **storybook.alfadocs.com**.
