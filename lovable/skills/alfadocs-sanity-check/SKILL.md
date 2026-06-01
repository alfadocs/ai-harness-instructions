---
name: alfadocs-sanity-check
description: Use when finishing an AlfaDocs app or before shipping, sharing, or demoing it — a fast end-to-end smoke check that it actually runs and connects (build, login, /me, one real API call, secrets server-side, kit UI, webhooks 200) before the deeper review skills. Run this first; it routes anything deeper to the matching review skill.
---

# AlfaDocs app — pre-ship sanity check

A **fast triage gate**, not a deep audit. Answer two questions: *does it actually
run and connect?* and *is anything obviously unsafe?* Anything that needs depth,
hand off to the matching review skill (listed at the end) — don't try to do their
job here.

Work top to bottom. The first failure usually explains the rest, so fix and re-run.

## 1. It runs

- [ ] **Build is clean** — `npm run build` (or the project's build) succeeds with **no TypeScript errors**.
- [ ] **App loads** — the dev/preview URL renders the first screen; no blank page, no error boundary.
- [ ] **Console is clean** — no uncaught errors or red warnings in the browser console on load and on the main flow.
- [ ] **No leaked logging** — no `console.log` of tokens, API responses, or PII anywhere in committed code (Standard §9, §10).

## 2. It connects (the AlfaDocs end-to-end path)

- [ ] **Auth completes** — OAuth "Sign in with AlfaDocs" returns to your callback and you land authenticated (or the API-key flow stores the key server-side and a call succeeds).
- [ ] **`/me` returns data** — the app calls `/me` and gets a real `practiceId` + `archiveId` back (not `401`, not empty).
- [ ] **One real call works** — at least one further AlfaDocs endpoint returns actual data using those IDs and the **one** auth header for the app's method (`Authorization: Bearer …` for OAuth, **or** `X-Api-Key: …` for an API key — never both), plus `Accept`.
- [ ] **Webhooks (if any) ack correctly** — the handler returns **HTTP 200** (never 202) and parses `application/x-www-form-urlencoded`, not JSON.

## 3. The show-stoppers (top blockers, fast scan)

- [ ] **Secrets are server-side** — API key / OAuth Client Secret live in **Supabase Edge Function secrets**, never in the frontend, a `VITE_*` var, or git. The browser never holds an AlfaDocs token (Standard §11).
- [ ] **Data is tenant-scoped** — every read/write is scoped by `practiceId`/`archiveId`; Supabase RLS is on and deny-by-default; handlers authorize the caller's IDs before any work (Standard §4, §11).
- [ ] **UI is the kit** — every screen is built from `@alfadocs/ui-kit`; no Material UI / Bootstrap / shadcn / Chakra / hand-rolled CSS (Standard §12).
- [ ] **CORS isn't `*`** — Edge Functions restrict CORS to known origins (Standard §8).

## Verdict

- **All boxes ticked →** safe to ship/demo. Optionally run the depth reviews below.
- **Any box failed →** it's not ready. Fix the failures first, then for anything
  beyond a quick fix, hand off:

| If this looked off | Load |
|---|---|
| Secrets, RLS, tokens, XSS/CSP, consent | `alfadocs-security-review` |
| Tenant scoping, webhook logic, token refresh | `alfadocs-logic-review` |
| BFF shape, Edge Function layout, multi-tenancy | `alfadocs-architecture-review` |
| Code quality, reuse, error handling | `alfadocs-code-review` |
| Design-system compliance | `alfadocs-design-review` |
| Keyboard, focus, contrast, RTL, labels | `alfadocs-accessibility-review` |

Keep this pass quick — its value is catching the show-stopper before a full review,
not replacing one.
