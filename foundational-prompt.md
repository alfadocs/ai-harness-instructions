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
- The browser never holds an access token. Tokens live only in an httpOnly cookie set by the Edge Function.
- There is NO frontend auth client. The Edge Function is the entire auth system.

What to build first (in this order):

1. A Supabase Edge Function at supabase/functions/alfadocs-auth/index.ts.
   Use @alfadocs/oauth2-client. The library handles everything server-side:
   login redirect, OAuth2 PKCE callback, session validation, token refresh, and logout.

   npm install github:alfadocs/oauth2-client

   import { createAlfadocsSupabaseAuth } from '@alfadocs/oauth2-client'

   const auth = createAlfadocsSupabaseAuth({
     clientId:         Deno.env.get("ALFADOCS_CLIENT_ID")!,
     clientSecret:     Deno.env.get("ALFADOCS_CLIENT_SECRET")!,
     redirectUri:      Deno.env.get("ALFADOCS_REDIRECT_URI")!,
     appOrigin:        Deno.env.get("ALLOWED_ORIGIN")!,
     baseUrl:          "https://app.alfadocs.com",
     appPostLoginPath: "/dashboard",
   })

   Deno.serve((req) => auth.handleRequest(req))

   The single handleRequest() method routes all auth traffic automatically:
   - GET ?action=login   → starts login, redirects to AlfaDocs
   - GET ?code=...       → handles OAuth2 callback, sets cookie, redirects to /dashboard
   - GET (no params)     → returns { authenticated: boolean, user?: object }
   - POST               → clears session cookie (logout)
   - OPTIONS            → CORS preflight

2. A ProtectedRoute component at src/components/ProtectedRoute.tsx.
   It fetches the session from the Edge Function — no auth client needed on the frontend:

   const SESSION_URL = `${import.meta.env.VITE_SUPABASE_URL}/functions/v1/alfadocs-auth`

   export function ProtectedRoute({ children }) {
     const [status, setStatus] = useState('loading')
     useEffect(() => {
       fetch(SESSION_URL, { credentials: 'include' })
         .then(r => r.json())
         .then(({ authenticated }) => setStatus(authenticated ? 'ok' : 'unauth'))
         .catch(() => setStatus('unauth'))
     }, [])
     if (status === 'loading') return <div>Loading…</div>   // always show a spinner
     if (status === 'unauth') return <Navigate to="/login" replace />
     return children
   }

3. Two routes only (there is no /callback page — the Edge Function handles the callback):
   - /login  → link or button pointing to:
               `${VITE_SUPABASE_URL}/functions/v1/alfadocs-auth?action=login`
               Do NOT use window.location.href from JS — use a plain <a> tag or redirect.
   - All other routes → wrapped in ProtectedRoute

4. Supabase secrets (set via Lovable's Secrets panel, NOT with VITE_ prefix):
   ALFADOCS_CLIENT_ID
   ALFADOCS_CLIENT_SECRET
   ALFADOCS_REDIRECT_URI   → the Edge Function URL: https://<project>.supabase.co/functions/v1/alfadocs-auth
   ALLOWED_ORIGIN          → https://yourapp.lovable.app

5. Vite env vars (safe to expose, VITE_ prefix is OK):
   VITE_SUPABASE_URL       → your Supabase project URL

Hard rules:
- Do not use supabase.auth for AlfaDocs login.
- Do not store tokens in localStorage, sessionStorage, React state, or any browser-accessible location.
- Do not call AlfaDocs APIs from the browser.
- Always use credentials: "include" on every fetch to Supabase Edge Functions.
- Never use Access-Control-Allow-Origin: * — the library sets this from ALLOWED_ORIGIN automatically.
- Never expose error details (stack traces, error messages) in any custom Edge Function responses.
- package.json must have a "build" script. The CI pipeline runs npm run build and fails without it.
- Never put any key other than SUPABASE_URL / SUPABASE_ANON_KEY / SUPABASE_PUBLISHABLE_KEY /
  SUPABASE_PROJECT_ID (and their VITE_ equivalents) inside any .env file. Any other key in a .env
  file will fail the CI pipeline's environment policy check.

Once auth works end-to-end (login → Edge Function callback → session check → logout), then build:
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
- Does ProtectedRoute fetch the session from the Edge Function with credentials: "include"?
  → It must call GET /functions/v1/alfadocs-auth and check { authenticated }.
- Is there a visible loading state while the session check is in flight?
  → Never render protected content before the check completes.
- Does a false/unauthenticated response redirect immediately to /login?
  → Must redirect, not silently fail.
- Is there a /callback page in the React app?
  → Remove it. The Edge Function handles the OAuth2 callback server-side.
  → ALFADOCS_REDIRECT_URI must point to the Edge Function URL, not a frontend route.
- Is the login action a link/redirect to the Edge Function with ?action=login?
  → GET ${VITE_SUPABASE_URL}/functions/v1/alfadocs-auth?action=login

Check 3 — Secrets and environment variables:
- Are any secrets or credentials accidentally using the VITE_ prefix?
  → VITE_ bakes the value into the browser bundle. Move to Supabase secrets.
- Are any keys other than SUPABASE_URL / SUPABASE_ANON_KEY / SUPABASE_PUBLISHABLE_KEY /
  SUPABASE_PROJECT_ID (and their VITE_ equivalents) inside any .env file?
  → The CI pipeline will fail. Move those keys to Supabase secrets.
- Does .gitignore include .env and .env.*?
  → Add it if missing.

Check 4 — Session cookie:
- Does the Edge Function set the session cookie with all required attributes?
  HttpOnly, Secure, SameSite=None, Path=/, Max-Age set.
  → The library handles this automatically. Only check if using a custom cookie setup.
- Is SameSite=Lax or SameSite=Strict used anywhere?
  → Replace with SameSite=None; Secure.

Check 5 — CORS:
- Is Access-Control-Allow-Origin set to * anywhere?
  → Replace with the exact ALLOWED_ORIGIN env var value.
- Is Access-Control-Allow-Credentials: "true" present on all Edge Function responses?
  → Required for cookies to be accepted by the browser.

Check 6 — Edge Function security (custom Edge Functions only, not alfadocs-auth):
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

Check 8 — CI pipeline compliance:
- Does package.json have a "build" script?
  → The pipeline runs npm run build and fails if it is missing or broken.
- Is there a lockfile committed (package-lock.json, bun.lockb, yarn.lock, or pnpm-lock.yaml)?
  → Required. The pipeline installs with --frozen-lockfile.
- Does any .env file contain keys outside the Supabase allowlist?
  → Only SUPABASE_URL / SUPABASE_ANON_KEY / SUPABASE_PUBLISHABLE_KEY / SUPABASE_PROJECT_ID
     (and their VITE_ equivalents) are allowed in committed .env files.
- Has any secret ever been committed to the git history, even in a later-removed commit?
  → Gitleaks scans the full git history. A secret in any past commit will fail the pipeline.

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

3. Any VITE_ prefix on any secret or credential (including VITE_ALFADOCS_CLIENT_SECRET)
   → VITE_ means it's baked into the browser bundle. Move all secrets to Supabase secrets.

4. Any use of initAlfadocsAuth or the old frontend auth client from @alfadocs/oauth2-client
   → Replace entirely with the server-side library on the Edge Function:
     npm install github:alfadocs/oauth2-client
     import { createAlfadocsSupabaseAuth } from '@alfadocs/oauth2-client'
   → Delete any src/auth.ts frontend auth client file.

5. Any supabase.auth usage for AlfaDocs login
   → Remove. Use createAlfadocsSupabaseAuth in the Edge Function instead.

6. Any /callback React page that calls auth.handleCallback()
   → Remove the /callback page entirely. Set ALFADOCS_REDIRECT_URI to the Edge Function URL.
     The library handles the callback server-side and redirects the browser to appPostLoginPath.

7. Any session cookie with SameSite=Lax or SameSite=Strict
   → Must be SameSite=None; Secure (Lovable and Supabase are on different domains).

8. Any CORS config using Access-Control-Allow-Origin: *
   → Must be the exact frontend URL from the ALLOWED_ORIGIN env var.

9. Any fetch() to Supabase that is missing credentials: "include"
   → Add it. Without it, the session cookie is not sent and auth breaks.

10. Any missing session check on protected pages, or protected content rendered before the
    check completes
    → Add a ProtectedRoute wrapper with a loading spinner. It must fetch:
      GET ${VITE_SUPABASE_URL}/functions/v1/alfadocs-auth with credentials: "include"
      and check { authenticated }.

11. Any Edge Function that reads the Authorization header from the browser request and
    forwards it to AlfaDocs
    → Read the token from the session cookie server-side instead.

12. Any Edge Function request body used without Zod validation
    → Add validation before any business logic runs.

13. Any .env file containing keys beyond the Supabase allowlist
    → Remove them. Only SUPABASE_URL / SUPABASE_ANON_KEY / SUPABASE_PUBLISHABLE_KEY /
      SUPABASE_PROJECT_ID (and their VITE_ equivalents) are allowed.

14. Missing "build" script in package.json, or missing lockfile
    → Add the script and commit the lockfile. The CI pipeline requires both.

For each issue found, show me what you changed and why before moving on.
Once the audit is clean, we can continue with new features.
```

---

## FOR LOVABLE: Must-Have Rules

> These are the 16 non-negotiable rules for this project. They apply to every file, every edit, every feature. Memorize them. Never violate them.

1. **Never call AlfaDocs APIs from React or the browser.** All AlfaDocs API calls go through Supabase Edge Functions. The browser only talks to Supabase.

2. **Never store tokens in the browser.** No localStorage, sessionStorage, React state, Zustand, Redux, context, globals, or anywhere else. Tokens live only in the httpOnly session cookie.

3. **Always use `credentials: "include"` on every fetch to Supabase.** Without it the session cookie is never sent and auth silently breaks.

4. **Always check the session from the Edge Function before rendering protected content.** `ProtectedRoute` must `GET /functions/v1/alfadocs-auth` with `credentials: "include"` and read `{ authenticated }`. Show a loading spinner while in flight. Redirect to `/login` if false.

5. **Never flash protected content before the session check completes.** Always show a loading state. Render nothing auth-sensitive until the check resolves.

6. **Never use the `VITE_` prefix on secrets.** `VITE_` variables are baked into the browser bundle at build time. Secrets belong in Supabase secrets only.

7. **Session cookie must have all five attributes: `HttpOnly; Secure; SameSite=None; Path=/; Max-Age`.** The library sets these automatically. If you write a custom cookie, all five are required.

8. **Never use `Access-Control-Allow-Origin: *`.** Always use the exact origin from `ALLOWED_ORIGIN`. Wildcards disable cookie-based auth entirely.

9. **Always include `Access-Control-Allow-Credentials: "true"` on every Edge Function response.** Required for the browser to accept the session cookie.

10. **Validate every custom Edge Function request body with Zod before using it.** No exceptions. Parse first, then act.

11. **Never expose error details in Edge Function responses.** Return a generic message. Log the full error server-side only.

12. **Never build AlfaDocs URLs from user input or request headers.** Use hardcoded path templates with `Deno.env.get("ALFADOCS_BASE_URL")` as the base only.

13. **Read tokens from the session cookie server-side. Never forward the browser's `Authorization` header to AlfaDocs.** The browser does not hold tokens and must never send them.

14. **Add `X-Requested-With: XMLHttpRequest` to every non-GET fetch from the browser, and verify it in every corresponding Edge Function.** This is required CSRF protection for all mutations.

15. **Use `@alfadocs/oauth2-client` and never implement auth logic manually.** Install with `npm install github:alfadocs/oauth2-client`. Use `createAlfadocsSupabaseAuth` in the Edge Function. There is no frontend auth client.

16. **Never put non-Supabase keys in `.env` files, never commit secrets to git history, and always have a working `npm run build` script.** The CI pipeline enforces all three and will block deployment on any violation.

---

## FOR LOVABLE: Development Rules (Always Apply)

> You are a senior software engineer. Apply every rule below on every edit, even if the user's prompt does not mention security.

---

### Architecture

```
Browser → Supabase Edge Function (alfadocs-auth) → AlfaDocs API
```

- The React frontend never calls AlfaDocs APIs directly. Ever.
- All AlfaDocs auth and API calls happen inside Supabase Edge Functions (Deno runtime).
- The browser communicates only with Supabase Edge Functions, always with `credentials: "include"`.
- There is no frontend auth client. The Edge Function is the entire auth system.
- Do not use `supabase.auth` for AlfaDocs authentication — it is a different system.

---

### Authentication

Install the library in the Edge Function project:
```bash
npm install github:alfadocs/oauth2-client
```

Set up the Edge Function at `supabase/functions/alfadocs-auth/index.ts`:
```ts
import { createAlfadocsSupabaseAuth } from '@alfadocs/oauth2-client'

const auth = createAlfadocsSupabaseAuth({
  clientId:         Deno.env.get("ALFADOCS_CLIENT_ID")!,
  clientSecret:     Deno.env.get("ALFADOCS_CLIENT_SECRET")!,
  redirectUri:      Deno.env.get("ALFADOCS_REDIRECT_URI")!,   // the Edge Function URL itself
  appOrigin:        Deno.env.get("ALLOWED_ORIGIN")!,
  baseUrl:          "https://app.alfadocs.com",
  appPostLoginPath: "/dashboard",                              // where to send user after login
  cookieMaxAgeSeconds: 604800,                                 // 7 days; match your token TTL
})

Deno.serve((req) => auth.handleRequest(req))
```

The `handleRequest()` method routes all auth traffic via query params and HTTP method — no custom routing needed:

| Signal | Handler | What it does |
|---|---|---|
| `OPTIONS *` | `handleOptions` | CORS preflight → 204 |
| `GET ?action=login` | `handleLogin` | Generates PKCE pair + state, redirects to AlfaDocs |
| `GET ?code=...` | `handleCallback` | Exchanges code, stores session cookie, redirects to app |
| `GET` (no params) | `handleSession` | Returns `{ authenticated, user? }`, auto-refreshes token |
| `POST` | `handleLogout` | Clears session cookie → `{ ok: true }` |

`ALFADOCS_REDIRECT_URI` must be set to the Edge Function URL itself (e.g., `https://<project>.supabase.co/functions/v1/alfadocs-auth`), not a frontend route. The library redirects the browser back to `appOrigin + appPostLoginPath` after a successful callback.

#### Protected Route

There is no `/callback` page. Wrap every non-public route in `ProtectedRoute`:
```tsx
// src/components/ProtectedRoute.tsx
import { useEffect, useState } from 'react'
import { Navigate } from 'react-router-dom'

const SESSION_URL = `${import.meta.env.VITE_SUPABASE_URL}/functions/v1/alfadocs-auth`

export function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const [status, setStatus] = useState<'loading' | 'ok' | 'unauth'>('loading')

  useEffect(() => {
    fetch(SESSION_URL, { credentials: 'include' })
      .then(r => r.json())
      .then(({ authenticated }) => setStatus(authenticated ? 'ok' : 'unauth'))
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

#### Login and Logout

```tsx
// SESSION_URL is already defined in ProtectedRoute.tsx — reuse it
// Login — plain link, no JS auth logic needed
function LoginButton() {
  return <a href={`${SESSION_URL}?action=login`}>Sign in with AlfaDocs</a>
}

// Logout — POST to the Edge Function, then redirect
async function logout(navigate: (path: string) => void) {
  await fetch(SESSION_URL, {
    method: 'POST',
    credentials: 'include',
    headers: { 'X-Requested-With': 'XMLHttpRequest' },
  })
  navigate('/login')
}
```

Never implement PKCE, token exchange, state verification, or cookie logic manually. The library handles all of it.

---

### Environment Variables

`VITE_` prefix = baked into the browser bundle at build time = visible to anyone in DevTools.

| Variable | Where | VITE_ prefix? | Why |
|---|---|---|---|
| `VITE_SUPABASE_URL` | Frontend | ✅ OK | Public Supabase project URL |
| `VITE_SUPABASE_ANON_KEY` | Frontend | ✅ OK | Publishable anon key — scope with RLS |
| `ALFADOCS_CLIENT_ID` | Edge Function | ❌ Never in VITE_ | Supabase secrets only |
| `ALFADOCS_CLIENT_SECRET` | Edge Function | ❌ Never | Supabase secrets only |
| `ALFADOCS_REDIRECT_URI` | Edge Function | ❌ Never | Supabase secrets only |
| `ALLOWED_ORIGIN` | Edge Function | ❌ Never | Supabase secrets only |
| Any token, API key, or credential | — | ❌ Never | Supabase secrets only |

#### .env File Allowlist (CI Pipeline-Enforced)

The pipeline validates every `.env`, `.env.local`, `.env.development`, `.env.production`, `.env.test`, and `.env.staging` file. **Only these 8 keys are permitted:**

```
SUPABASE_URL
SUPABASE_PUBLISHABLE_KEY
SUPABASE_PROJECT_ID
SUPABASE_ANON_KEY
VITE_SUPABASE_URL
VITE_SUPABASE_PUBLISHABLE_KEY
VITE_SUPABASE_PROJECT_ID
VITE_SUPABASE_ANON_KEY
```

Any other key in a committed `.env` file causes an immediate pipeline failure. All other configuration goes in Supabase secrets. `.env.example` is not validated and may document any keys as placeholders.

Always provide `.env.example` with placeholder values. Never commit `.env`. Always include `.env*` in `.gitignore`.

---

### Session Cookie

The library sets the session cookie automatically with the correct attributes:
```
HttpOnly        — JS cannot read it
Secure          — HTTPS only
SameSite=None   — required: Lovable (lovable.app) and Supabase (supabase.co) are different origins
Path=/
Max-Age         — configured via cookieMaxAgeSeconds (default 7 days)
```

If you write any custom cookie in your own Edge Functions, apply all the same attributes.
Never use `SameSite=Lax` or `SameSite=Strict` — they silently break cross-origin cookie sending.
`SameSite=None` requires `Secure` — browsers reject `SameSite=None` cookies without it.

---

### CORS

The `alfadocs-auth` Edge Function handles CORS automatically via the library. For any **custom** Edge Functions you add:

```ts
const corsHeaders = {
  "Access-Control-Allow-Origin":      Deno.env.get("ALLOWED_ORIGIN")!,  // never *
  "Access-Control-Allow-Credentials": "true",                            // required for cookies
  "Access-Control-Allow-Methods":     "GET, POST, OPTIONS",
  "Access-Control-Allow-Headers":     "Content-Type, X-Requested-With",
}

// Always handle preflight
if (req.method === "OPTIONS") {
  return new Response(null, { status: 204, headers: corsHeaders })
}
```

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
| httpOnly cookie via Edge Function | ✅ Only here | — |

---

### API Proxy Pattern

Every custom Supabase Edge Function that calls AlfaDocs must:
1. Read the session cookie and extract the access token — never from a request header
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
console.error("AlfaDocs API call failed:", e)          // server-side log
return new Response(JSON.stringify({ error: "Request failed" }), { status: 500, headers: corsHeaders })

// ❌ Wrong
return new Response(JSON.stringify({ error: e.message, stack: e.stack }), { status: 500 })
```

Never proxy AlfaDocs error responses verbatim to the browser — always map to a generic message.

---

### CSRF Protection

The OAuth2 `state` parameter and PKCE (handled automatically by the library) protect the auth flow.

For **all** POST/PATCH/DELETE calls from the browser to Supabase, send `X-Requested-With` and verify it — this is required, not optional:
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

Include on **all** custom Edge Function responses:
```ts
const securityHeaders = {
  "X-Content-Type-Options":  "nosniff",
  "X-Frame-Options":         "DENY",
  "Referrer-Policy":         "strict-origin-when-cross-origin",
  "Permissions-Policy":      "camera=(), microphone=(), geolocation=()",
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

- `ALFADOCS_REDIRECT_URI` must be the Edge Function URL — never a frontend route or URL parameter
- After login, the Edge Function redirects to `appOrigin + appPostLoginPath` — never to user-supplied URLs
- After logout, only redirect to known internal routes — never to external URLs

---

### React Security

- Never use `dangerouslySetInnerHTML` unless sanitized with DOMPurify first
- Never interpolate user input or API data into `href`, `src`, or `style` without validation
- Never use `eval()` or `new Function(string)` with dynamic content
- Always add `rel="noopener noreferrer"` to `<a target="_blank">` links
- Never `console.log` tokens, cookies, session objects, or anything from the auth flow
- Treat AlfaDocs API responses as untrusted — validate shape before rendering

---

### Pipeline Guardrails (Automated — Applied to Every Push)

The CI/CD pipeline runs six automated checks on every push to `main`/`master`. A failure on any check blocks deployment and triggers an auto-fix PR. Write code that passes all six from the start:

**1. ESLint + eslint-plugin-security** — Fails on: `eval()` with dynamic expressions, non-literal regular expressions, non-literal `require()`, child_process with variable arguments, Unicode bidi characters (Trojan source), disabled Mustache escaping. All security rules are treated as errors, not warnings.

**2. Semgrep (OWASP Top 10)** — Semantic security analysis covering injection, broken auth, sensitive data exposure, XSS, insecure deserialization, and more. JavaScript, TypeScript, and Node.js rulesets all run.

**3. Gitleaks (full git history scan)** — Scans every commit ever made, not just the latest. A secret committed and later removed in a subsequent commit is still in the history and will still fail. There is no recovery from this without a force-push history rewrite. **Never commit a secret, even temporarily.**

**4. npm audit** — Scans for high-severity dependency vulnerabilities. If found and no Dependabot config exists, the pipeline creates one and marks the build failed to trigger a fix. Keep dependencies updated.

**5. .env file allowlist** — Only the 8 Supabase keys listed in the Environment Variables section are permitted in committed `.env*` files. Any other key causes immediate failure.

**6. Build verification** — `npm run build` must exist in `package.json` and must succeed. The build runs with a frozen lockfile — commit your lockfile and do not modify `package.json` in ways that invalidate it.

Branch convention: guardrails run on `main`/`master` pushes. Only the `stable` branch is deployed. Promotion to stable happens automatically when guardrails pass.

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
| `initAlfadocsAuth(...)` or any frontend auth client | Wrong API — use `createAlfadocsSupabaseAuth` in the Edge Function |
| A `/callback` React page that handles the OAuth2 exchange | Edge Function handles it server-side |
| Rendering protected content before session check resolves | Auth bypass / content flicker |
| Unversioned Deno imports (`deno.land/x/zod/mod.ts`) | Supply chain risk |
| Missing `credentials: "include"` on fetch to Supabase | Cookie not sent, auth silently breaks |
| Missing `X-Requested-With` on mutating fetches | CSRF vulnerability |
| `SameSite=None` without `Secure` | Browser rejects the cookie |
| Missing `Access-Control-Allow-Credentials: "true"` | Browser rejects the cookie |
| Auth redirect target from URL param or user input | Open redirect vulnerability |
| Any non-Supabase key in a committed `.env` file | CI pipeline env policy failure |
| Missing `npm run build` script in package.json | CI pipeline build check failure |
| Committing any secret to git history, even temporarily | Gitleaks scans full history — irrecoverable |
| Modifying package.json without updating the lockfile | CI pipeline uses --frozen-lockfile |