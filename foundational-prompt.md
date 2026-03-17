# Alfadocs Foundational Guidelines

This document has two purposes:
1. **Prompt templates for PMs** — paste into Lovable when starting, maintaining, or securing a project.
2. **Persistent rules for Lovable** — always apply these rules on every edit, not just at project start.

---

## FOR PMs: Starting a New Project

> Paste the block below into Lovable as your **first message**, then describe your app underneath it.

---

```
I'm building a Lovable app that integrates with AlfaDocs for authentication.
Before writing any feature code, set up the full authentication and session flow first.

Architecture rules (non-negotiable):
- This is a Backend-for-Frontend (BFF) app. The React frontend never calls AlfaDocs APIs directly.
- All AlfaDocs API calls go through Supabase Edge Functions.
- The browser never holds an access token. Tokens live only in an httpOnly cookie set by Supabase.

What to build first (in this order):

1. A Supabase Edge Function at supabase/functions/alfadocs-auth/index.ts that handles:
   - POST / → exchanges the OAuth2 code for tokens, sets an httpOnly session cookie
   - GET /session → validates the cookie, returns { authenticated: boolean }, auto-refreshes if expired
   - POST /logout → clears the cookie
   Use the exact Edge Function template from the @alfadocs/oauth2-client README.

2. A React auth client at src/auth.ts using @alfadocs/oauth2-client:
   npm install github:alfadocs/oauth2-client#v0.2.1

   import { initAlfadocsAuth } from '@alfadocs/oauth2-client'
   export const auth = initAlfadocsAuth({
     clientId:    import.meta.env.VITE_ALFADOCS_CLIENT_ID,
     redirectUri: `${window.location.origin}/callback`,
     exchangeUrl: `${import.meta.env.VITE_SUPABASE_URL}/functions/v1/alfadocs-auth`,
     baseUrl:     'https://app.alfadocs.com',
   })

3. A ProtectedRoute component at src/components/ProtectedRoute.tsx:
   - Calls auth.refreshSession() on mount
   - Shows a loading spinner while the session check is in flight
   - Redirects to /login if refreshSession() returns false
   - Only renders children after confirming session is valid
   Never flash protected content before the session check completes.

4. Three routes:
   - /login → calls auth.login() on button click
   - /callback → calls auth.handleCallback() on mount, shows loading spinner, then navigates to dashboard
   - All other routes → wrapped in ProtectedRoute

5. Supabase secrets (set via Lovable's Secrets panel, NOT with VITE_ prefix):
   ALFADOCS_CLIENT_ID, ALFADOCS_CLIENT_SECRET, ALFADOCS_REDIRECT_URI,
   ALFADOCS_BASE_URL=https://app.alfadocs.com, ALLOWED_ORIGIN

6. Vite env vars (safe to expose, VITE_ prefix is OK):
   VITE_ALFADOCS_CLIENT_ID, VITE_SUPABASE_URL

Hard rules:
- Do not use supabase.auth for AlfaDocs login.
- Do not store tokens in localStorage, sessionStorage, React state, or any browser-accessible location.
- Do not call AlfaDocs APIs from the browser.
- Always use credentials: "include" on every fetch to Supabase Edge Functions.
- Never use Access-Control-Allow-Origin: * — always use the exact ALLOWED_ORIGIN env var.
- Always validate Edge Function request bodies with Zod before using them.
- Never expose error details (stack traces, error messages) in Edge Function responses.

Once auth works end-to-end (login → callback → session check → logout), then build:
[DESCRIBE YOUR APP FEATURES HERE]
```

---

## FOR PMs: Ongoing Development Check

> Paste this at any point during development to keep the project compliant. Run it before adding new features, after a big refactor, or whenever something feels off.

---

```
Before we continue, do a full Alfadocs compliance check on everything built so far.
Review all existing code and fix every violation you find. Report what you changed.

Check 1 — Architecture:
- Are there any fetch() calls to app.alfadocs.com from React or browser code?
  → Move to Supabase Edge Functions. The browser must only call Supabase.
- Do all fetches from React to Supabase include credentials: "include"?
  → Add it to every call. Without it the session cookie is not sent.

Check 2 — Authentication:
- Are all protected pages and routes wrapped in ProtectedRoute (or equivalent)?
  → Every non-public route must check session before rendering content.
- Is there a visible loading state while auth.refreshSession() is in flight?
  → Never render protected content before the session check completes.
- Does auth.refreshSession() failure redirect to /login?
  → A false return must immediately redirect, not silently fail.
- Is auth.handleCallback() called on the /callback page on mount?
  → Without it the OAuth2 code is never exchanged and login breaks.

Check 3 — Secrets and environment variables:
- Are any secrets or credentials accidentally using the VITE_ prefix?
  → VITE_ bakes the value into the browser bundle. Move to Supabase secrets.
- Are ALFADOCS_CLIENT_SECRET and any API keys set as Supabase secrets only?
  → They must never appear in .env files committed to the repo.
- Does .gitignore include .env and .env.*?
  → Add it if missing.

Check 4 — Session cookie:
- Does the Supabase Edge Function set the cookie with all required attributes?
  HttpOnly, Secure, SameSite=None, Path=/, Max-Age set.
  → Fix any missing attribute. SameSite=None requires Secure.
- Is SameSite=Lax or SameSite=Strict used anywhere?
  → Replace with SameSite=None; Secure.

Check 5 — CORS:
- Is Access-Control-Allow-Origin set to * anywhere?
  → Replace with the exact ALLOWED_ORIGIN env var value.
- Is Access-Control-Allow-Credentials: "true" present on all Edge Function responses?
  → Required for cookies to be accepted by the browser.

Check 6 — Edge Function security:
- Are all request bodies validated with Zod before use?
  → Add validation to any Edge Function that reads req.json().
- Are AlfaDocs URLs built from user input or request headers?
  → Must use hardcoded templates with env var base URL only (SSRF risk).
- Do Edge Functions read tokens from the session cookie, not from request Authorization headers?
  → The browser must never send tokens directly.
- Do Edge Function error responses expose internal details?
  → Only return generic messages. Log full errors server-side.
- Do all Edge Functions include security headers?
  X-Content-Type-Options: nosniff, X-Frame-Options: DENY, Referrer-Policy: strict-origin-when-cross-origin
- Do POST/PATCH/DELETE Edge Functions check X-Requested-With: XMLHttpRequest?
  → Add the check if missing (CSRF protection).

Check 7 — React security:
- Is dangerouslySetInnerHTML used anywhere without DOMPurify sanitization?
  → Add DOMPurify or remove entirely.
- Are there any console.log() calls that print tokens, cookies, or session data?
  → Remove them all.
- Are external links using target="_blank" without rel="noopener noreferrer"?
  → Add the rel attribute.

Fix everything found before continuing with new features.
```

---

## FOR PMs: Security Audit for Existing Projects

> Use this once if you have an existing Lovable project that was built without these rules and needs a full migration.

---

```
I need you to audit and fix this project to follow the Alfadocs security architecture.

Audit checklist — find and fix every instance of:

1. Any fetch() to AlfaDocs APIs (app.alfadocs.com) from the browser or React code
   → Move those calls to Supabase Edge Functions. The browser only calls Supabase.

2. Any access_token or refresh_token stored in localStorage, sessionStorage, React state,
   Zustand, Redux, or any browser-accessible location
   → Tokens must only live in the httpOnly session cookie set by the Supabase Edge Function.

3. Any VITE_ALFADOCS_CLIENT_SECRET or VITE_ prefix on any secret or credential
   → VITE_ means it's baked into the browser bundle. Move all secrets to Supabase secrets.

4. Any supabase.auth usage for AlfaDocs login
   → Replace entirely with @alfadocs/oauth2-client (npm install github:alfadocs/oauth2-client#v0.2.1).

5. Any session cookie with SameSite=Lax or SameSite=Strict
   → Must be SameSite=None; Secure (Lovable and Supabase are on different domains).

6. Any CORS config using Access-Control-Allow-Origin: *
   → Must be the exact frontend URL from the ALLOWED_ORIGIN env var.

7. Any fetch() to Supabase that is missing credentials: "include"
   → Add it. Without it, the session cookie is not sent and auth breaks.

8. Any missing auth.refreshSession() call on app mount, or protected pages that render
   content before the session check completes
   → Add ProtectedRoute wrapper with a loading spinner.

9. Any Edge Function that reads the Authorization header from the browser request and
   forwards it to AlfaDocs
   → Read the token from the session cookie server-side instead.

10. Any Edge Function request body used without Zod validation
    → Add validation before any business logic runs.

For each issue found, show me what you changed and why before moving on.
Once the audit is clean, we can continue with new features.
```

---

## FOR LOVABLE: Must-Have Rules

> These are the 15 non-negotiable rules for this project. They apply to every file, every edit, every feature. Memorize them. Never violate them.

1. **Never call AlfaDocs APIs from React or the browser.** All AlfaDocs API calls go through Supabase Edge Functions. The browser only talks to Supabase.

2. **Never store tokens in the browser.** No localStorage, sessionStorage, React state, Zustand, Redux, context, globals, or anywhere else. Tokens live only in the httpOnly session cookie.

3. **Always use `credentials: "include"` on every fetch to Supabase.** Without it the session cookie is never sent and auth silently breaks.

4. **Always call `auth.refreshSession()` before rendering protected content.** Use a `ProtectedRoute` wrapper that shows a loading spinner while the check is in flight and redirects to `/login` if it returns false.

5. **Never flash protected content before the session check completes.** Always show a loading state. Render nothing auth-sensitive until `refreshSession()` resolves.

6. **Never use the `VITE_` prefix on secrets.** `VITE_` variables are baked into the browser bundle at build time. Secrets belong in Supabase secrets only.

7. **Session cookie must have all five attributes: `HttpOnly; Secure; SameSite=None; Path=/; Max-Age=604800`.** Missing any one of these breaks auth or security.

8. **Never use `Access-Control-Allow-Origin: *`.** Always use the exact origin from the `ALLOWED_ORIGIN` env var. Wildcards disable cookie-based auth entirely.

9. **Always include `Access-Control-Allow-Credentials: "true"` on every Edge Function response.** Required for the browser to accept the session cookie.

10. **Validate every Edge Function request body with Zod before using it.** No exceptions. Parse first, then act.

11. **Never expose error details in Edge Function responses.** Return a generic message. Log the full error server-side only.

12. **Never build AlfaDocs URLs from user input or request headers.** Use hardcoded path templates with `Deno.env.get("ALFADOCS_BASE_URL")` as the base only.

13. **Read tokens from the session cookie server-side. Never forward the browser's `Authorization` header to AlfaDocs.** The browser does not hold tokens and must never send them.

14. **Add `X-Requested-With: XMLHttpRequest` to every non-GET fetch from the browser, and verify it in every corresponding Edge Function.** This is required CSRF protection for all mutations.

15. **Use `@alfadocs/oauth2-client` with a pinned version tag. Never implement auth logic manually.** Install with `npm install github:alfadocs/oauth2-client#v0.2.1` and use only the five provided methods.

---

## FOR LOVABLE: Development Rules (Always Apply)

> You are a senior software engineer. Apply every rule below on every edit, even if the user's prompt does not mention security.

---

### Architecture

```
Browser → Supabase Edge Function → AlfaDocs API
```

- The React frontend never calls AlfaDocs APIs directly. Ever.
- All AlfaDocs API calls happen inside Supabase Edge Functions (Deno runtime).
- The browser communicates only with Supabase Edge Functions, always with `credentials: "include"`.
- Do not use `supabase.auth` for AlfaDocs authentication — it is a different system.

---

### Authentication

Install with a pinned version tag:
```bash
npm install github:alfadocs/oauth2-client#v0.2.1
```

Initialize once in `src/auth.ts`:
```ts
import { initAlfadocsAuth } from '@alfadocs/oauth2-client'

export const auth = initAlfadocsAuth({
  clientId:    import.meta.env.VITE_ALFADOCS_CLIENT_ID,
  redirectUri: `${window.location.origin}/callback`,
  exchangeUrl: `${import.meta.env.VITE_SUPABASE_URL}/functions/v1/alfadocs-auth`,
  baseUrl:     'https://app.alfadocs.com',
})
```

Use exactly these 5 methods — never implement auth logic manually:

| Method | Where | What it does |
|---|---|---|
| `auth.login()` | Login button | Redirects to AlfaDocs login |
| `auth.handleCallback()` | `/callback` page on mount | Exchanges code, sets cookie |
| `auth.refreshSession()` | `ProtectedRoute` on mount | Returns `true` if session is valid |
| `auth.isAuthenticated()` | Guards, conditional renders | Sync boolean, in-memory only |
| `auth.logout()` | Logout button | Clears cookie via Supabase |

#### Protected Route

Wrap every non-public route in a `ProtectedRoute` component. Never skip it:
```tsx
// src/components/ProtectedRoute.tsx
import { useEffect, useState } from 'react'
import { Navigate } from 'react-router-dom'
import { auth } from '@/auth'

export function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const [status, setStatus] = useState<'loading' | 'ok' | 'unauth'>('loading')

  useEffect(() => {
    auth.refreshSession()
      .then(ok => setStatus(ok ? 'ok' : 'unauth'))
      .catch(() => setStatus('unauth'))
  }, [])

  if (status === 'loading') return <div>Loading…</div>   // never skip this
  if (status === 'unauth') return <Navigate to="/login" replace />
  return <>{children}</>
}
```

Use it in your router:
```tsx
<Route path="/dashboard" element={<ProtectedRoute><Dashboard /></ProtectedRoute>} />
```

#### Callback Page

The `/callback` page must show a spinner while the exchange is in flight:
```tsx
useEffect(() => {
  auth.handleCallback()
    .then(() => navigate('/dashboard'))
    .catch(() => navigate('/login'))
}, [])

return <div>Completing login…</div>
```

Never render any app content on the callback page. Never navigate before `handleCallback()` resolves.

---

### Environment Variables

`VITE_` prefix = baked into the browser bundle at build time = visible to anyone in DevTools.

| Variable | VITE_ prefix? | Why |
|---|---|---|
| `VITE_ALFADOCS_CLIENT_ID` | ✅ OK | Appears in the OAuth2 authorize URL anyway |
| `VITE_SUPABASE_URL` | ✅ OK | Public Supabase project URL |
| `VITE_SUPABASE_ANON_KEY` | ✅ OK | Publishable anon key — safe to expose, but scope with RLS |
| `ALFADOCS_CLIENT_SECRET` | ❌ Never | Supabase secrets only |
| Any token, API key, or credential | ❌ Never | Supabase secrets only |

Always provide `.env.example` with placeholder values. Never commit `.env`. Always include `.env*` in `.gitignore`.

---

### Session Cookie

The Supabase Edge Function must set the cookie with **all** of these attributes:
```
HttpOnly        — JS cannot read it
Secure          — HTTPS only
SameSite=None   — required: Lovable (lovable.app) and Supabase (supabase.co) are different origins
Path=/
Max-Age=604800  — 7 days; adjust to match your token TTL
```

Never use `SameSite=Lax` or `SameSite=Strict` — they silently break cross-origin cookie sending.
`SameSite=None` requires `Secure` — browsers reject `SameSite=None` cookies without it.

---

### CORS

```ts
const corsHeaders = {
  "Access-Control-Allow-Origin":      Deno.env.get("ALLOWED_ORIGIN")!,  // never *
  "Access-Control-Allow-Credentials": "true",                            // required for cookies
  "Access-Control-Allow-Methods":     "GET, POST, OPTIONS",
  "Access-Control-Allow-Headers":     "Content-Type, X-Requested-With",
}
```

Always handle `OPTIONS` preflight requests and return `corsHeaders` with a `204` response.
Never use `Access-Control-Allow-Origin: *` — it disables cookie-based auth entirely.
`Access-Control-Allow-Credentials: true` requires a specific non-wildcard origin.

---

### Token & Auth State Storage

| Location | Tokens | Auth state |
|---|---|---|
| `localStorage` | ❌ Never | ❌ Never |
| `sessionStorage` | ❌ Never | ❌ Never |
| React state / Context / Zustand | ❌ Never | ✅ Boolean flag only (e.g. `isReady`) |
| `window.*` or globals | ❌ Never | ❌ Never |
| httpOnly cookie via Supabase | ✅ Only here | — |

---

### API Proxy Pattern

Every Supabase Edge Function that calls AlfaDocs must:
1. Read the token from the session cookie — never from a request header
2. Return `401` immediately if no valid token is found
3. Add `Authorization: Bearer {token}` server-side
4. Never forward any `Authorization` header from the browser

```ts
// ✅ Correct
const token = getAccessTokenFromCookie(req)
if (!token) return new Response("Unauthorized", { status: 401, headers: corsHeaders })

const res = await fetch(`${Deno.env.get("ALFADOCS_BASE_URL")}/api/v1/...`, {
  headers: { Authorization: `Bearer ${token}` },
})
```

```ts
// ❌ Wrong — forwards browser-supplied header
const res = await fetch(url, {
  headers: { Authorization: req.headers.get("Authorization") },
})
```

`ALFADOCS_BASE_URL` must always come from `Deno.env.get("ALFADOCS_BASE_URL")` — never hardcoded or from user input.

---

### Input Validation (Edge Functions)

Validate every request body with Zod (pinned version) before any business logic:
```ts
import { z } from 'https://deno.land/x/zod@3.23.8/mod.ts'

const body = z.object({ code: z.string().min(1).max(512) }).parse(await req.json())
```

Never construct AlfaDocs URLs from user-supplied values (SSRF prevention):

```ts
// ✅ Safe — URL built from env var + hardcoded template
const url = `${Deno.env.get("ALFADOCS_BASE_URL")}/api/v1/practices/${practiceId}/patients`
```

```ts
// ❌ Unsafe — SSRF risk
const targetUrl = req.headers.get("x-target-url")
```

Always use pinned URLs for Deno imports — never unversioned:

```ts
// ✅ Pinned
import { z } from 'https://deno.land/x/zod@3.23.8/mod.ts'
```

```ts
// ❌ Unversioned — supply chain risk
import { z } from 'https://deno.land/x/zod/mod.ts'
```

---

### Error Handling

Never expose internal details to the browser:
```ts
// ✅ Correct
console.error("AlfaDocs token exchange failed:", e)          // server-side log
return new Response(JSON.stringify({ error: "Request failed" }), { status: 500, headers: corsHeaders })

// ❌ Wrong
return new Response(JSON.stringify({ error: e.message, stack: e.stack }), { status: 500 })
```

Never proxy AlfaDocs error responses verbatim to the browser — always map to a generic message.

---

### CSRF Protection

The OAuth2 `state` parameter (handled automatically by the library) protects the auth flow.

For **all** POST/PATCH/DELETE calls through Supabase, send `X-Requested-With` and verify it — this is required, not optional:
```ts
// Browser — every mutating fetch
fetch(url, {
  method: "POST",
  credentials: "include",
  headers: {
    "Content-Type": "application/json",
    "X-Requested-With": "XMLHttpRequest",   // required
  },
  body: JSON.stringify(data),
})

// Edge Function — every mutating handler
if (req.method !== "GET" && req.method !== "OPTIONS") {
  if (req.headers.get("x-requested-with") !== "XMLHttpRequest") {
    return new Response("Forbidden", { status: 403, headers: corsHeaders })
  }
}
```

---

### Security Headers

Include on **all** Edge Function responses:
```ts
const securityHeaders = {
  "X-Content-Type-Options":  "nosniff",
  "X-Frame-Options":         "DENY",
  "Referrer-Policy":         "strict-origin-when-cross-origin",
}
```

In `index.html`:
```html
<meta http-equiv="Content-Security-Policy"
  content="default-src 'self';
           connect-src 'self' https://*.supabase.co;
           script-src 'self';
           style-src 'self' 'unsafe-inline';
           img-src 'self' data: https:;
           frame-ancestors 'none'">
```

---

### Redirect URI Security

- `redirectUri` must always be `${window.location.origin}/callback` — never from a URL parameter or user input
- After login or logout, only redirect to known internal routes — never to external URLs
- The Edge Function must verify `redirect_uri` matches `ALFADOCS_REDIRECT_URI` from env exactly

---

### React Security

- Never use `dangerouslySetInnerHTML` unless sanitized with DOMPurify first
- Never interpolate user input or API data into `href`, `src`, or `style` without validation
- Never use `eval()` or `new Function(string)` with dynamic content
- Always add `rel="noopener noreferrer"` to `<a target="_blank">` links
- Never `console.log` tokens, cookies, session objects, or anything from the auth flow
- Treat AlfaDocs API responses as untrusted — validate shape before rendering

---

### Prohibited Patterns — Never Do These

| Pattern | Why |
|---|---|
| `VITE_ALFADOCS_CLIENT_SECRET=...` | Exposed in browser bundle |
| `fetch("https://app.alfadocs.com/api/...")` from React | Token would need to be in the browser |
| `localStorage.setItem("token", ...)` | Stolen by XSS |
| `Access-Control-Allow-Origin: *` | Breaks credentials, insecure |
| `SameSite=Lax` or `SameSite=Strict` on session cookie | Silently breaks cross-origin auth |
| `dangerouslySetInnerHTML={{ __html: apiData }}` | XSS |
| `console.log(token)` or `console.log(session)` | Token leaks into browser logs |
| `return new Response(JSON.stringify({ error: e.message }))` | Leaks internals |
| URL built from user input in Edge Function | SSRF |
| Forwarding browser `Authorization` header to AlfaDocs | Token injection |
| `supabase.auth.*` for AlfaDocs login | Wrong auth system |
| Rendering protected content before `refreshSession()` resolves | Auth bypass / flicker |
| Unversioned Deno imports (`deno.land/x/zod/mod.ts`) | Supply chain risk |
| Missing `credentials: "include"` on fetch to Supabase | Cookie not sent, auth silently breaks |
| Missing `X-Requested-With` on mutating fetches | CSRF vulnerability |
| `SameSite=None` without `Secure` | Browser rejects the cookie |
| Missing `Access-Control-Allow-Credentials: "true"` | Browser rejects the cookie |
| Auth redirect target from URL param or user input | Open redirect vulnerability |