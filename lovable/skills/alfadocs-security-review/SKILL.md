---
name: alfadocs-security-review
description: Use when reviewing or auditing the security of an AlfaDocs app before going live — checking that secrets stay server-side, no AlfaDocs token reaches the browser, Supabase RLS is deny-by-default and practice-scoped, no XSS/CSP holes, OAuth callback matches exactly, webhooks aren't blindly trusted, and patient data has consent. Load this for a security pass, not for building.
---

# AlfaDocs app — security audit

> **Enforces Engineering Standard** (workspace knowledge) §5, §8, §11. Cite the section number when you flag a violation.

A pre-ship security review for an AlfaDocs app (Lovable frontend + Supabase backend, calling `https://app.alfadocs.com/api/v1`). Work the checklist top to bottom. **Fix every blocker before the app goes live; flag warnings and notes.** This reinforces the build-time guardrails in Workspace Knowledge and the `alfadocs-connected-app-oauth` / `alfadocs-connected-app-api-key` / `alfadocs-webhooks` / `alfadocs-gdpr-consent` skills — never contradict them.

The grep/scan patterns for each section live in [audit-checks.md](./audit-checks.md). Run them; don't eyeball.

---

## 1. Secrets are server-side only — *blocker*

The OAuth **Client Secret**, any **API key**, and any third-party credential give full access to a practice's data. They live **only** in Supabase Edge Function secrets, read via `Deno.env.get(...)`.

- [ ] No `ALFADOCS_CLIENT_SECRET`, `ALFADOCS_API_KEY`, `ALFADOCS_CLIENT_ID`, or third-party key in any `VITE_*` variable, `.env`, or committed file. `VITE_*` is baked into the public bundle — anything there is published.
- [ ] **Env allowlist holds.** The only keys allowed in any `.env*` are the eight public Supabase ones (`SUPABASE_URL`, `SUPABASE_PUBLISHABLE_KEY`, `SUPABASE_PROJECT_ID`, `SUPABASE_ANON_KEY` and their `VITE_` twins). Anything else is a finding.
- [ ] **Scan git history, not just the working tree** — a secret committed yesterday and deleted today is still leaked. If found, the fix is to **rotate the secret upstream** (regenerate it) and treat the old value as public; deleting the file is not enough.
- [ ] Frontend holds only `VITE_SUPABASE_URL` (+ the public Supabase keys). It never needs the Client ID, Client Secret, or API key.

## 2. The browser never holds an AlfaDocs token — *blocker*

Every call to `https://app.alfadocs.com/api/v1` goes through a Supabase Edge Function. The access token stays server-side, keyed by an `HttpOnly` cookie.

- [ ] No `fetch("https://app.alfadocs.com/api/...")` from React. AlfaDocs calls come from an Edge Function proxy only.
- [ ] No access/refresh token in `localStorage`, `sessionStorage`, a non-`HttpOnly` cookie, or any `VITE_*`.
- [ ] Edge Function proxies return only what the frontend needs — never the raw token.
- [ ] For OAuth apps: login is the single `alfadocs-auth` Edge Function (BFF). No OAuth library in the browser, no React `/callback` page, no `supabase.auth.signIn(...)` used to log into AlfaDocs.
- [ ] Token refresh runs server-side with a concurrency lock; one practice's token is never used against another practice's endpoint (see `alfadocs-oauth-token-refresh`).

## 3. Supabase RLS — deny-by-default, practice-scoped — *blocker*

AlfaDocs data is multi-tenant per `practiceId` (+ `archiveId`). A leak across practices is the worst-case failure.

- [ ] Every table holding tenant/AlfaDocs data has RLS **enabled**.
- [ ] A restrictive policy denies `anon` and `authenticated` outright — only Edge Functions (service role) touch these tables. No table is left open to `anon`/`authenticated`.
- [ ] Every such table carries `practice_id` (and usually `archive_id`); every query inside Edge Functions is scoped by it. No cross-practice read, write, or join.
- [ ] The frontend can never choose an arbitrary `practiceId`/`archiveId`. The Edge Function derives them from `/me` (or the stored session), so the browser can't reach another practice's data.
- [ ] Webhook-target tables stay locked to `anon`/`authenticated` too (the webhook request is unauthenticated by RLS — service role only inside the function).

## 4. XSS & Content-Security-Policy — *blocker for sinks, warning for CSP gaps*

- [ ] No raw `dangerouslySetInnerHTML`. The **only** sanctioned path for rendered HTML is `SafeHtml`.
- [ ] No `eval`, `new Function`, dynamic `require`, or user-controlled URL/path/regex construction (these are the OWASP-style sinks the build guardrails reject).
- [ ] User/API/webhook data is not injected into a non-React layer (direct DOM, `innerHTML`) unsanitised.
- [ ] Strictest practical **CSP** is shipped. Remove `https://lovable.dev` from `img-src`. Avoid `'unsafe-inline'` / `'unsafe-eval'` in `script-src` where avoidable; no `*` source lists for script/connect.
- [ ] OG/Twitter preview images are AlfaDocs-branded, not the Lovable defaults.

## 5. OAuth callback & origin hardening — *blocker*

- [ ] The registered Callback URL **byte-matches** `ALFADOCS_REDIRECT_URI` exactly (scheme, host, path, trailing slash) — and points at the Edge Function's `/callback`. One character off breaks the flow. One registered per environment.
- [ ] `ALLOWED_ORIGIN` is the exact frontend origin — **never `"*"`**. CORS and `Access-Control-Allow-Origin` are scoped to it.
- [ ] `state` is validated on callback (CSRF defence on the OAuth flow); logout verifies `Origin`.
- [ ] Mutating Edge Function requests require `X-Requested-With: XMLHttpRequest` and reject when it's absent.
- [ ] Session cookie is `HttpOnly` + `Secure` + an appropriate `SameSite`; frontend `fetch` sends `credentials: "include"`.

## 6. Dismissable-layer integrity — *warning*

A subtle but real class of bug that can trap users inside modals/dialogs over patient data.

- [ ] No global capture-phase `document.addEventListener("pointerdown"/"mousedown", …, true)` that calls `stopPropagation()`. It silently breaks ui-kit overlays (Dialog, Popover, DropdownMenu, Tooltip) that rely on outside-press to dismiss — leaving consent/confirm layers stuck open. Use the component's own dismiss handlers instead.

## 7. Webhooks aren't blindly trusted — *blocker for trust, warning for hygiene*

The webhook endpoint is public and unauthenticated by Supabase RLS.

- [ ] The handler verifies the request before acting on it (shared/signing secret from an Edge Function secret, or equivalent), rather than processing any POST that arrives.
- [ ] It resolves the practice from the **`archiveId`** in the payload against your stored mapping, then tenant-scopes all work by the resolved `practiceId` — it never trusts a `practiceId` sent in the body.
- [ ] Body parsed as `application/x-www-form-urlencoded` (indexed bracket notation), not JSON; handler **always responds HTTP 200**, never 202 (correctness, but a wrong status also drives retry storms).
- [ ] An unknown/missing `archiveId` is logged and skipped — still 200; it is not used to write under an arbitrary tenant.

## 8. No token / PII logging — *blocker*

- [ ] No `console.log` / error log emits an access token, refresh token, Client Secret, or API key.
- [ ] No patient/personal data (name, DOB, contact, fiscal code/tax ID, clinical notes) is logged where it isn't strictly necessary; logs that must reference a patient use an ID, not identifying fields.
- [ ] Errors returned to the browser don't leak internal tokens, secrets, or raw upstream error bodies.

## 9. Explicit consent before processing patient data — *blocker*

Per `alfadocs-gdpr-consent`: no patient/personal data leaves the local browser session (to AlfaDocs-adjacent storage, a third party, or your Supabase DB) without explicit, logged consent.

- [ ] Consent checkbox on **every** page/section that can send data out; not pre-ticked; outgoing-action buttons disabled until ticked.
- [ ] Consent is logged append-only to Supabase from an Edge Function (user + practice/archive + timestamp + fingerprint). Revoke sets `revoked_at`; it never deletes prior rows.
- [ ] A server-side consent check runs before **every** outgoing data call — the browser checkbox state alone is not trusted.
- [ ] Data minimisation: only the fields the feature needs are sent, only for the stated purpose. No real practice/patient data is connected to a prototype during dev.

---

## How to report findings

Produce one merged punchlist, grouped by severity, each item citing the **file/area** (and line where possible):

- **Blocker** — must fix before going live: any leaked or client-side secret, a token reachable by the browser, RLS missing / open to `anon`/`authenticated` / not practice-scoped, a raw XSS sink, a mismatched callback URL or `"*"` origin, an unverified webhook, token/PII in logs, or patient data processed without server-checked consent. **Fix these as part of the review.**
- **Warning** — should fix: weak-but-present CSP, capture-phase pointerdown breaking overlays, webhook 200/parsing hygiene, broad logging, missing `X-Requested-With` guard.
- **Note** — informational: hardening suggestions, defence-in-depth, doc gaps.

State explicitly whether the app is **clear to go live**. If there are zero blockers, say so. If there are blockers, list each with file/area and the concrete fix, and do not sign off.
