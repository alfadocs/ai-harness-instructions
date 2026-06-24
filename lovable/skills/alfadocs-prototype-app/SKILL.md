---
name: alfadocs-prototype-app
description: Use when building a quick INTERNAL mockup or prototype of an AlfaDocs UI — no backend, no auth, no AlfaDocs API, just on-brand screens built with @alfadocs/ui-kit and inline mock data. NOT for a real connected app — for that use alfadocs-staff-sso-app (internal staff tool), alfadocs-connected-app-oauth (customer marketplace product), or alfadocs-connected-app-api-key (single-practice API-key tool).
---

# Build an AlfaDocs prototype / mockup (no backend)

This is for a **throwaway, on-brand UI prototype or mockup** — for internal review, design exploration, or a quick demo. Speed and looking right are the whole point. There is **no backend, no auth, and no AlfaDocs API**: every value is inline mock data.

If the thing you're building needs real users, real data, or real AlfaDocs access, **stop — this is the wrong archetype**:

| If… | Use |
|---|---|
| Quick internal mockup / prototype, no real data (this) | **`alfadocs-prototype-app`** |
| Internal staff tool — employees sign in, own data | `alfadocs-staff-sso-app` |
| Customer product — many practices "Sign in with AlfaDocs" | `alfadocs-connected-app-oauth` |
| One external practice, no login, API key | `alfadocs-connected-app-api-key` |

## What to do

- **Build all UI with `@alfadocs/ui-kit`** so the mockup looks like real AlfaDocs — follow `alfadocs-design-system` for the wiring (import `@alfadocs/ui-kit/tokens` once, wrap in `ThemeRoot` + `I18nextProvider` + `TooltipProvider` + `Toaster`, use `MarketplaceAppShell` for the app chrome). Browse components at **storybook.alfadocs.cloud** and reach for the dedicated primitive (DataTable, Card, Dialog, EmptyState, …) rather than hand-rolling.
- **Mock data is inline**, in plain TS fixtures, each clearly marked `// mock — replace before shipping`. Use realistic-but-fake content — **never real customer names and never real PHI**.
- Strings via `useTranslation('app')`; `var(--…)` kit tokens only (no inline styles, no raw hex); logical properties.

## What NOT to do

- **No Supabase, no Edge Functions, no `.env`, no secrets** — a prototype has no backend. Don't add `@alfadocs/auth`, an API key, or any `fetch` to `app.alfadocs.com`.
- **No real auth.** If you want a "logged-in" look, fake it with a static user in `MarketplaceAppShell` — don't wire a real login.
- Don't paste real practice/patient data in to make it look convincing.

## When the prototype graduates

When a mockup needs to become a real app, **start fresh from the right archetype** — remix/begin from `alfadocs-staff-sso-app` (internal staff tool) or `alfadocs-connected-app-oauth` (marketplace product) and port your screens in. Don't bolt auth + a backend onto a prototype ad-hoc.

## Quick checklist

- [ ] All UI from `@alfadocs/ui-kit`; chrome via `MarketplaceAppShell`; no other UI library.
- [ ] Data is inline mock fixtures marked "mock — replace before shipping"; no real names/PHI.
- [ ] No Supabase / Edge Functions / `.env` / secrets / `@alfadocs/auth` / AlfaDocs API calls.
- [ ] Strings via `useTranslation('app')`; kit tokens only; logical properties; strict TS.
- [ ] If it needs real data/auth → switch to `alfadocs-staff-sso-app` or `alfadocs-connected-app-oauth`.
