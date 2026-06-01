---
name: alfadocs-architecture-review
description: Use when reviewing or auditing the architecture, structure, or system design of an AlfaDocs app (Lovable + Supabase) — checking the BFF shape, where secrets and tokens live, whether the browser ever calls AlfaDocs directly, Edge Function layout, Supabase RLS, practiceId multi-tenancy, and session handling — before shipping or when asked "is this built correctly".
---

# AlfaDocs app — architecture review

> **Enforces Engineering Standard** (workspace knowledge) §1 (layer direction), §4 and §11 (tenancy / BFF). Cite the section number when you flag a violation.

Audit the **shape** of an AlfaDocs app: who holds the secrets, who talks to AlfaDocs, how tenants stay isolated, how sessions work. This is a structural review, not a styling or feature review. Read the code top-down before judging — the violations are usually about *where* a call lives, not whether it works.

The one rule everything else falls out of: **the browser is untrusted and cannot keep secrets.** Any AlfaDocs token, client secret, or API key reachable from React is exfiltratable by an XSS bug, a browser extension, or DevTools. So all of it lives server-side in Supabase Edge Functions, and React only ever talks to those functions.

## How to run this review

1. List the Edge Functions (`supabase/functions/*`) and the frontend `fetch`/network calls.
2. Trace one data path end-to-end: React → Edge Function → AlfaDocs → back. Note where the token enters and where it must not.
3. Walk the checklist below. For each item, cite the file/area and mark blocker / warning / note.
4. Fix blockers; flag warnings and notes for the author.

See [checklist.md](./checklist.md) for the full item-by-item rubric with what "wrong" looks like.

## The checklist (summary)

**BFF boundary — the browser never touches AlfaDocs**
- [ ] No `fetch("https://app.alfadocs.com/...")` anywhere in `src/` (React). Every AlfaDocs call goes through an Edge Function. **Blocker.**
- [ ] No AlfaDocs access token, refresh token, client secret, or API key in the bundle, in `VITE_*`, in `.env`, in `localStorage`/`sessionStorage`, or in React state/Context. **Blocker.**
- [ ] Secrets read only via `Deno.env.get(...)` inside Edge Functions. **Blocker if elsewhere.**
- [ ] The frontend's only Supabase-related env is the function base URL (`VITE_SUPABASE_URL`) — never a credential. **Blocker if a secret is `VITE_`-prefixed.**

**Function layout — one auth function, separate data functions**
- [ ] Auth lives in a single `alfadocs-auth` Edge Function (login + callback + session + logout). No second `*-callback` function; no `/callback` page in React. **Blocker.**
- [ ] Data access lives in **separate** Edge Functions, not bolted onto the auth function. Clear separation of concerns. **Warning.**
- [ ] No duplicated proxy logic — shared helpers (session lookup, AlfaDocs headers, token refresh) are reused, not copy-pasted per function. Reuse existing functions over adding near-identical ones. **Warning.**

**Sessions & tokens**
- [ ] Session is an **opaque** value in an `HttpOnly; Secure; SameSite=None; Path=/` cookie; tokens are stored server-side keyed by that cookie. The cookie carries no token. **Blocker if a token is in the cookie/body/response.**
- [ ] React sends `credentials: "include"` on every BFF fetch, and checks `/session` before rendering protected content (no flash of protected UI). **Warning** (blocker if it leaks data).
- [ ] Token refresh stays server-side and is concurrency-safe (row-level lock); one practice's token is never used to serve another. **Blocker** for cross-practice token reuse.

**Multi-tenancy — practiceId end-to-end**
- [ ] `practiceId` (and `archiveId`) come from a server-side `/me` lookup, **not** from a frontend-supplied value the browser can tamper with. **Blocker.**
- [ ] Every persisted row that mirrors AlfaDocs data carries its `practiceId`; every query filters by it. No cross-practice read/write/join is reachable. **Blocker.**

**RLS — defence in depth**
- [ ] RLS is **enabled** on every table holding practice data, and it is **deny-by-default**: the `anon` and `authenticated` roles have no policy granting access. Only the service role (Edge Functions) touches the tables. **Blocker if a table is open to `anon`/`authenticated` or RLS is off.**
- [ ] RLS is treated as a backstop, not the only guard — the Edge Function still enforces `practiceId`. RLS being on does not excuse a missing tenant filter. **Warning.**

**Hygiene**
- [ ] Custom Edge Functions validate request bodies (Zod or equiv.) before use; reject mutating requests without `X-Requested-With: XMLHttpRequest`; CORS is `ALLOWED_ORIGIN` exactly (never `*`) with `Allow-Credentials: true`. **Warning** (blocker if it enables CSRF or origin bypass).
- [ ] AlfaDocs URLs are built from env base + hardcoded route templates, never from user input (SSRF). Tokens never logged. Internal errors not echoed to the client. **Blocker for SSRF / token logging.**

## Reporting findings

Report a single merged punchlist, grouped by severity, each item citing the **file or area**:

- **Blocker** — breaks the security model or tenant isolation: any AlfaDocs call from the browser, any secret/token reachable client-side, a token in the cookie/response, RLS off or open to `anon`/`authenticated`, frontend-chosen `practiceId`, cross-practice token reuse, SSRF, token logging. **Fix these before the app ships.**
- **Warning** — should fix: data logic mixed into the auth function, duplicated proxy code, missing `credentials: "include"`, missing CSRF/Origin/CORS hardening, RLS on but tenant filter missing in the function.
- **Note** — informational: structural simplifications, naming, opportunities to consolidate functions.

If there are **zero blockers**, say so explicitly and confirm the architecture is sound. List blockers first, with file + line where you can, and fix them; leave warnings and notes for the author to action.
