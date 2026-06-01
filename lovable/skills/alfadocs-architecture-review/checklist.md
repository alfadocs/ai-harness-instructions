# Architecture review — full rubric

Item-by-item detail behind the summary in `SKILL.md`. Each section says what to look for, what "wrong" looks like, and the default severity. Severity can rise if the flaw actually exposes data or tokens.

## 1. BFF boundary — the browser never touches AlfaDocs

The React frontend talks **only** to Supabase Edge Functions. Those functions are the only thing that talks to AlfaDocs and to the database. This is non-negotiable because the browser cannot keep a secret: anything `fetch`-able from React is exfiltratable by XSS, a malicious extension, or a user with DevTools — and AlfaDocs tokens grant access to clinical data.

Look for:

- `fetch("https://app.alfadocs.com/...")` or any axios/SDK call to `app.alfadocs.com` inside `src/`. **Blocker** — the token would have to be in the browser to make this call.
- Any AlfaDocs access token, refresh token, OAuth **Client Secret**, or **API key** that the browser can read: in the built bundle, in a `VITE_*` env var, in a committed `.env`, in `localStorage`/`sessionStorage`, or in React state/Context/global. **Blocker.**
- The `VITE_` prefix on any AlfaDocs credential. Vite bakes `VITE_*` into the production bundle — it is effectively published. **Blocker.**
- Secrets read anywhere other than `Deno.env.get(...)` inside an Edge Function. **Blocker.**

The frontend legitimately knows only the **function base URL** (`VITE_SUPABASE_URL`) — never the client ID, client secret, API key, or any token.

### What "wrong" looks like

| Anti-pattern | Why it breaks |
|---|---|
| `VITE_ALFADOCS_CLIENT_SECRET=...` / `VITE_ALFADOCS_API_KEY=...` | Baked into the bundle; public |
| `fetch("https://app.alfadocs.com/api/...")` from React | Token would have to live in the browser |
| `localStorage.setItem("token", ...)` | Stolen by any XSS |
| Token in a response body the React app reads | Defeats the whole point of the BFF |

## 2. Function layout — one auth function, separate data functions

**Auth is one function.** The entire login system is a single `alfadocs-auth` Edge Function handling `/login`, `/callback`, `/session`, `/logout`. There is no OAuth library in the browser and no `/callback` page in React — the callback is server-side.

- A second `*-callback` function, or a React `/callback` route. **Blocker** — the callback exchanges the code+secret for tokens and must be server-side.
- For API-key apps (single practice), the equivalent is one proxy Edge Function holding `ALFADOCS_API_KEY`; same principle.

**Data access is separate.** Custom data proxies live in their own Edge Functions, not piled into the auth function. Each does one job: read the session, look up the token server-side, call AlfaDocs with the required headers, return only what the frontend needs. **Warning** if data logic is entangled with the auth function.

**Reuse over duplication.** Session lookup, the AlfaDocs auth header (`Authorization: Bearer` for OAuth **or** `X-Api-Key` for API-key, plus `Accept: application/json`), and token refresh should be shared helpers, not copy-pasted into every proxy. Before approving a new function, check an existing one doesn't already cover it. **Warning** for near-identical duplicated proxies.

## 3. Sessions & tokens

- The session cookie is **opaque** and `HttpOnly; Secure; SameSite=None; Path=/`. Tokens are stored server-side (Supabase / app DB) keyed by that cookie. The cookie itself carries no token. A token in the cookie, body, or any response → **Blocker**.
- `SameSite=Lax` on the session cookie breaks the cross-origin auth flow; `SameSite=None` without `Secure` is rejected by the browser. **Warning** (blocker if it breaks auth or drops the `Secure` flag).
- React sends `credentials: "include"` on every BFF fetch (so the cookie travels) and checks `/session` **before** rendering protected content — no flash of protected UI before the check resolves. **Warning**, rising to **Blocker** if protected data actually renders pre-check.
- Token **refresh** stays server-side and is concurrency-safe (row-level lock so two requests don't race a refresh). Crucially, one practice's token is **never** used to serve another practice's request. Cross-practice token reuse → **Blocker**.

## 4. Multi-tenancy — practiceId end-to-end

AlfaDocs data is multi-tenant: it belongs to a **practice**, and one practice's data must never leak to another. Tenancy is keyed on `practiceId` (and `archiveId`) at every layer.

- `practiceId` + `archiveId` are resolved **server-side** from `GET /me`, then reused. The browser must never choose an arbitrary `practiceId`/`archiveId` — if a frontend request can name the tenant, it can reach another practice's data. **Blocker.**
- Every persisted row that mirrors AlfaDocs data carries the `practiceId` it belongs to; every read/write/join filters by it. A query that can return rows across practices → **Blocker**.
- Build single-practice (API-key) apps as if there could be more than one practice — carry and filter by `practiceId` anyway, so the isolation is structural, not incidental. **Note** (warning if absent and the app could grow).

## 5. RLS — defence in depth

Row Level Security is the backstop beneath the Edge Function tenant filter — it must be **deny-by-default**.

- RLS is **enabled** on every table holding practice data. RLS off on such a table → **Blocker**.
- The `anon` and `authenticated` roles are denied outright — no policy grants them access. Only the **service role** (Edge Functions) touches the tables. A table readable/writable by `anon` or `authenticated` → **Blocker**.
- RLS does not replace the application-layer check: the Edge Function still enforces `practiceId`. "RLS is on" is not an excuse for a missing tenant filter in the function. **Warning** if the function relies on RLS alone.

## 6. Hygiene that the BFF shape implies

These are smaller, but they follow directly from the BFF being the trust boundary:

- **Validate input.** Custom Edge Functions validate every request body (Zod or equivalent) before using it. **Warning.**
- **CSRF intent.** Mutating (non-`GET`) requests must carry `X-Requested-With: XMLHttpRequest` and the function must reject those that don't. **Warning**, rising to **Blocker** if it leaves a CSRF hole.
- **CORS.** `Access-Control-Allow-Origin` is `ALLOWED_ORIGIN` exactly (never `*`) with `Access-Control-Allow-Credentials: true`. `*` with credentials is also rejected by the browser. **Warning** (blocker if it allows an untrusted origin).
- **No SSRF.** AlfaDocs URLs are built from an env base URL + hardcoded route templates, never concatenated from user input. URL-from-user-input on the BFF → **Blocker**.
- **No token leakage.** Never `console.log` a token; never echo internal/AlfaDocs error detail to the client (log server-side, return a generic message). Token in logs → **Blocker**.
- **Never forward the browser's `Authorization` header** to AlfaDocs — read the token from the session server-side. Forwarding it is token injection. **Blocker.**

## Severity quick-reference

| Finding | Severity |
|---|---|
| AlfaDocs call from the browser | Blocker |
| Secret/token reachable client-side (`VITE_*`, `.env`, `localStorage`, bundle) | Blocker |
| Token in the session cookie / response body | Blocker |
| Second `*-callback` function or React `/callback` page | Blocker |
| `practiceId`/`archiveId` chosen by the frontend | Blocker |
| Query/join that crosses practices | Blocker |
| RLS off, or table open to `anon`/`authenticated` | Blocker |
| Cross-practice token reuse | Blocker |
| SSRF (URL from user input) or token logging | Blocker |
| Data logic inside the auth function | Warning |
| Duplicated proxy logic | Warning |
| Missing `credentials: "include"` / pre-check / CSRF / CORS hardening | Warning |
| Function relies on RLS alone, no app-layer tenant filter | Warning |
| Structural simplification, consolidation, naming | Note |
