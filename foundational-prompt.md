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

Architecture (non-negotiable):
- Backend-for-Frontend (BFF). The React frontend never calls AlfaDocs APIs directly.
- All AlfaDocs calls go through Supabase Edge Functions.
- The browser never holds a token. Tokens live only in an httpOnly cookie set by the Edge Function.
- There is NO frontend auth client. The Edge Function is the entire auth system.

What to build first (in this order):

1. Supabase Edge Function at supabase/functions/alfadocs-auth/index.ts:

   npm install github:alfadocs/oauth2-client

   import { createAlfadocsSupabaseAuth } from '@alfadocs/oauth2-client'
   const auth = createAlfadocsSupabaseAuth({
     clientId:         Deno.env.get("ALFADOCS_CLIENT_ID")!,
     clientSecret:     Deno.env.get("ALFADOCS_CLIENT_SECRET")!,
     redirectUri:      Deno.env.get("ALFADOCS_REDIRECT_URI")!,  // the Edge Function URL itself
     appOrigin:        Deno.env.get("ALLOWED_ORIGIN")!,
     baseUrl:          "https://app.alfadocs.com",
     appPostLoginPath: "/dashboard",
   })
   Deno.serve((req) => auth.handleRequest(req))

   handleRequest() routes everything: GET ?action=login → login, GET ?code=... → callback,
   GET → session check ({ authenticated, user? }), POST → logout, OPTIONS → CORS preflight.

2. ProtectedRoute at src/components/ProtectedRoute.tsx (no auth client on the frontend):

   const SESSION_URL = `${import.meta.env.VITE_SUPABASE_URL}/functions/v1/alfadocs-auth`

   export function ProtectedRoute({ children }) {
     const [status, setStatus] = useState('loading')
     useEffect(() => {
       fetch(SESSION_URL, { credentials: 'include' })
         .then(r => r.json())
         .then(({ authenticated }) => setStatus(authenticated ? 'ok' : 'unauth'))
         .catch(() => setStatus('unauth'))
     }, [])
     if (status === 'loading') return <div>Loading…</div>  // never skip the spinner
     if (status === 'unauth') return <Navigate to="/login" replace />
     return children
   }

3. Two routes only — no /callback page (the Edge Function handles the OAuth2 callback):
   - /login  → <a href={`${SESSION_URL}?action=login`}>Sign in</a>
   - All other routes → wrapped in <ProtectedRoute>

4. Supabase secrets (Lovable Secrets panel, NOT VITE_ prefix):
   ALFADOCS_CLIENT_ID, ALFADOCS_CLIENT_SECRET,
   ALFADOCS_REDIRECT_URI=https://<project>.supabase.co/functions/v1/alfadocs-auth,
   ALLOWED_ORIGIN=https://yourapp.lovable.app

5. Vite env vars (VITE_ prefix OK, safe to expose):
   VITE_SUPABASE_URL

Hard rules:
- Never store tokens in localStorage, sessionStorage, React state, or any browser location.
- Never call AlfaDocs APIs from the browser. Always go through the Edge Function.
- Always use credentials: "include" on every fetch to Supabase.
- Never use Access-Control-Allow-Origin: * — use ALLOWED_ORIGIN only.
- Never put any key except SUPABASE_URL / SUPABASE_ANON_KEY / SUPABASE_PUBLISHABLE_KEY /
  SUPABASE_PROJECT_ID (and their VITE_ variants) in any .env file — CI will fail.
- package.json must have a "build" script. Commit a lockfile. CI uses --frozen-lockfile.

Once auth works end-to-end (login → callback → session → logout), describe your features:
[DESCRIBE YOUR APP FEATURES HERE]
```

---

## FOR PMs: Ongoing Development Check

> Paste this at any point during development — before new features, after a big refactor, or whenever something feels off.

---

```
Do a full Alfadocs compliance check on everything built so far. Fix every violation. Report changes.

Architecture:
- Any fetch() to app.alfadocs.com from React? → Move to Supabase Edge Function.
- All Supabase fetches include credentials: "include"?

Authentication:
- All protected routes wrapped in ProtectedRoute?
- ProtectedRoute calls GET /functions/v1/alfadocs-auth with credentials: "include" and checks { authenticated }?
- Loading spinner shown while check is in flight? Protected content never flashes?
- A false response redirects immediately to /login?
- Is there a /callback page? → Remove it. ALFADOCS_REDIRECT_URI must point to the Edge Function URL.

Secrets and .env files:
- Any secret using VITE_ prefix? → Move to Supabase secrets.
- Any .env file with keys beyond SUPABASE_URL / SUPABASE_ANON_KEY / SUPABASE_PUBLISHABLE_KEY /
  SUPABASE_PROJECT_ID (and VITE_ variants)? → Remove — CI will fail.
- .env* in .gitignore?

Session cookie:
- Cookie has HttpOnly, Secure, SameSite=None, Path=/, Max-Age? (Library sets this automatically.)
- Any SameSite=Lax or SameSite=Strict? → Replace with SameSite=None; Secure.

CORS (custom Edge Functions only):
- Access-Control-Allow-Origin set to *? → Replace with ALLOWED_ORIGIN env var.
- Access-Control-Allow-Credentials: "true" on all responses?

Custom Edge Function security:
- All request bodies validated with Zod?
- Tokens read from session cookie, not from Authorization headers?
- AlfaDocs URLs built from user input? → Use only env var base + hardcoded path templates.
- Error responses expose internal details? → Return generic message, log server-side.
- Security headers on all responses? (X-Content-Type-Options, X-Frame-Options, Referrer-Policy)
- POST/PATCH/DELETE check X-Requested-With: XMLHttpRequest?

React security:
- dangerouslySetInnerHTML without DOMPurify? → Sanitize or remove.
- console.log() on tokens/cookies/session data? → Remove.
- target="_blank" without rel="noopener noreferrer"? → Add it.

CI pipeline:
- package.json has "build" script and lockfile committed?
- Has any secret ever been committed to git history, even if later removed?
  → Gitleaks scans full history. This fails the pipeline and cannot be undone without a force-push.

Fix everything before continuing.
```

---

## FOR PMs: Security Audit for Existing Projects

> Use this once to migrate an existing Lovable project to these security rules.

---

```
Audit and fix this project to follow the Alfadocs security architecture. For each issue, explain what you changed and why.

1. Any fetch() to app.alfadocs.com from React → move to Supabase Edge Function.
2. Tokens in localStorage / sessionStorage / React state / Zustand / Redux → remove. Tokens live only in the httpOnly session cookie.
3. VITE_ prefix on any secret → move to Supabase secrets.
4. initAlfadocsAuth or any frontend auth client → delete src/auth.ts. Replace with createAlfadocsSupabaseAuth in the Edge Function (npm install github:alfadocs/oauth2-client).
5. supabase.auth for AlfaDocs login → remove entirely.
6. /callback React page → delete it. Set ALFADOCS_REDIRECT_URI to the Edge Function URL. The library handles the callback server-side.
7. SameSite=Lax or SameSite=Strict on session cookie → replace with SameSite=None; Secure.
8. Access-Control-Allow-Origin: * → replace with ALLOWED_ORIGIN env var.
9. Supabase fetches missing credentials: "include" → add to every call.
10. Protected pages with no session check or that render before check completes → add ProtectedRoute with spinner.
11. Edge Function forwarding browser Authorization header to AlfaDocs → read token from session cookie instead.
12. Edge Function request bodies not validated with Zod → add validation.
13. .env files with non-Supabase keys → move to Supabase secrets.
14. Missing "build" script or lockfile → add both.

Once the audit is clean, we can continue with new features.
```

---

## FOR LOVABLE: Must-Have Rules

> 16 non-negotiable rules. Apply to every file, every edit, every feature. Never violate them.

1. **Never call AlfaDocs APIs from React or the browser.** All calls go through Supabase Edge Functions.
2. **Never store tokens in the browser.** Nowhere — not localStorage, sessionStorage, state, context, or globals. Tokens live only in the httpOnly session cookie.
3. **Always use `credentials: "include"` on every fetch to Supabase.** Without it the cookie is never sent.
4. **Always check session from the Edge Function before rendering protected content.** `ProtectedRoute` fetches `GET /functions/v1/alfadocs-auth` with `credentials: "include"`, reads `{ authenticated }`, shows a spinner while in flight, redirects to `/login` if false.
5. **Never flash protected content before the session check completes.**
6. **Never use the `VITE_` prefix on secrets.** `VITE_` variables are baked into the browser bundle. All secrets go in Supabase secrets only.
7. **Session cookie must have: `HttpOnly; Secure; SameSite=None; Path=/; Max-Age`.** The library sets these automatically. Required on any custom cookie too.
8. **Never use `Access-Control-Allow-Origin: *`.** Always use the exact value from `ALLOWED_ORIGIN`.
9. **Always include `Access-Control-Allow-Credentials: "true"` on every Edge Function response.**
10. **Validate every custom Edge Function request body with Zod before using it.** Parse first, then act.
11. **Never expose error details in Edge Function responses.** Return a generic message. Log full errors server-side only.
12. **Never build AlfaDocs URLs from user input or request headers.** Use `Deno.env.get("ALFADOCS_BASE_URL")` + hardcoded path templates only.
13. **Read tokens from the session cookie server-side. Never forward the browser's `Authorization` header to AlfaDocs.**
14. **Add `X-Requested-With: XMLHttpRequest` to every non-GET fetch from the browser, and verify it in the Edge Function.** Required CSRF protection for all mutations.
15. **Use `@alfadocs/oauth2-client` and never implement auth logic manually.** `createAlfadocsSupabaseAuth` in the Edge Function. There is no frontend auth client.
16. **Never put non-Supabase keys in `.env` files, never commit secrets to git history, always have `npm run build`.** The CI pipeline enforces all three and blocks deployment on any violation.

---

## FOR LOVABLE: Development Rules (Always Apply)

> You are a senior software engineer. Apply every rule below on every edit, even if the user's prompt does not mention security.

---

### Authentication

Install in the Edge Function project:
```bash
npm install github:alfadocs/oauth2-client
```

`supabase/functions/alfadocs-auth/index.ts`:
```ts
import { createAlfadocsSupabaseAuth } from '@alfadocs/oauth2-client'

const auth = createAlfadocsSupabaseAuth({
  clientId:            Deno.env.get("ALFADOCS_CLIENT_ID")!,
  clientSecret:        Deno.env.get("ALFADOCS_CLIENT_SECRET")!,
  redirectUri:         Deno.env.get("ALFADOCS_REDIRECT_URI")!,  // the Edge Function URL itself
  appOrigin:           Deno.env.get("ALLOWED_ORIGIN")!,
  baseUrl:             "https://app.alfadocs.com",
  appPostLoginPath:    "/dashboard",
  cookieMaxAgeSeconds: 604800,
})

Deno.serve((req) => auth.handleRequest(req))
```

Routing handled automatically by `handleRequest()`:

| Signal | What it does |
|---|---|
| `GET ?action=login` | PKCE + state → 302 redirect to AlfaDocs |
| `GET ?code=...` | Token exchange → session cookie → 302 to app |
| `GET` (no params) | Returns `{ authenticated, user? }`, auto-refreshes token |
| `POST` | Clears session cookie → `{ ok: true }` |
| `OPTIONS` | CORS preflight → 204 |

`ALFADOCS_REDIRECT_URI` = the Edge Function URL (not a frontend route). After callback, the library redirects to `appOrigin + appPostLoginPath`. After logout, redirect only to internal routes — never user-supplied URLs.

#### Protected Route

`src/components/ProtectedRoute.tsx` — wrap every non-public route here:
```tsx
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

  if (status === 'loading') return <div>Loading…</div>  // never skip this
  if (status === 'unauth') return <Navigate to="/login" replace />
  return <>{children}</>
}
```

#### Login and Logout

```tsx
// Login — plain link, no JS auth logic
function LoginButton() {
  return <a href={`${SESSION_URL}?action=login`}>Sign in with AlfaDocs</a>
}

// Logout
async function logout(navigate: (path: string) => void) {
  await fetch(SESSION_URL, {
    method: 'POST',
    credentials: 'include',
    headers: { 'X-Requested-With': 'XMLHttpRequest' },
  })
  navigate('/login')
}
```

Never implement PKCE, token exchange, state verification, or cookie logic manually.

---

### Environment Variables

| Variable | Where | VITE_ OK? |
|---|---|---|
| `VITE_SUPABASE_URL` | Frontend | ✅ public URL |
| `VITE_SUPABASE_ANON_KEY` | Frontend | ✅ publishable — scope with RLS |
| `ALFADOCS_CLIENT_ID` | Edge Function | ❌ Supabase secrets only |
| `ALFADOCS_CLIENT_SECRET` | Edge Function | ❌ Supabase secrets only |
| `ALFADOCS_REDIRECT_URI` | Edge Function | ❌ Supabase secrets only |
| `ALLOWED_ORIGIN` | Edge Function | ❌ Supabase secrets only |

**CI .env allowlist** — only these keys are permitted in `.env`, `.env.local`, `.env.development`, `.env.production`, `.env.test`, `.env.staging`. Any other key causes immediate pipeline failure:
```
SUPABASE_URL  SUPABASE_PUBLISHABLE_KEY  SUPABASE_PROJECT_ID  SUPABASE_ANON_KEY
VITE_SUPABASE_URL  VITE_SUPABASE_PUBLISHABLE_KEY  VITE_SUPABASE_PROJECT_ID  VITE_SUPABASE_ANON_KEY
```

Always provide `.env.example` (not validated, document any keys here). Never commit `.env`. Include `.env*` in `.gitignore`.

---

### Session Cookie

The library sets the cookie correctly. If you write a custom cookie, all attributes are required:
```
HttpOnly; Secure; SameSite=None; Path=/; Max-Age=604800
```
Never use `SameSite=Lax` or `SameSite=Strict` — they silently break cross-origin cookie sending. `SameSite=None` requires `Secure`.

---

### CORS (Custom Edge Functions Only)

```ts
const corsHeaders = {
  "Access-Control-Allow-Origin":      Deno.env.get("ALLOWED_ORIGIN")!,  // never *
  "Access-Control-Allow-Credentials": "true",
  "Access-Control-Allow-Methods":     "GET, POST, OPTIONS",
  "Access-Control-Allow-Headers":     "Content-Type, X-Requested-With",
}
if (req.method === "OPTIONS") return new Response(null, { status: 204, headers: corsHeaders })
```

---

### API Proxy Pattern

Every custom Edge Function that calls AlfaDocs must read the token from the session cookie — never from request headers:

```ts
// ✅ Correct — token from cookie, added server-side
const token = getAccessTokenFromCookie(req)
if (!token) return new Response("Unauthorized", { status: 401, headers: corsHeaders })
const res = await fetch(`${Deno.env.get("ALFADOCS_BASE_URL")}/api/v1/...`, {
  headers: { Authorization: `Bearer ${token}` },
})
// Log errors server-side; return only generic messages to the browser
// ✅ return new Response(JSON.stringify({ error: "Request failed" }), { status: 500, headers: corsHeaders })
// ❌ return new Response(JSON.stringify({ error: e.message, stack: e.stack }), { status: 500 })
```

```ts
// ❌ Wrong — forwards browser-supplied header, ALFADOCS_BASE_URL from user input
const res = await fetch(req.headers.get("x-target-url"), {
  headers: { Authorization: req.headers.get("Authorization") },
})
```

---

### Input Validation (Edge Functions)

```ts
import { z } from 'https://deno.land/x/zod@3.23.8/mod.ts'  // always pin Deno imports
// ❌ import { z } from 'https://deno.land/x/zod/mod.ts'   // supply chain risk

const body = z.object({ code: z.string().min(1).max(512) }).parse(await req.json())

// ✅ URL from env var + hardcoded template
const url = `${Deno.env.get("ALFADOCS_BASE_URL")}/api/v1/practices/${practiceId}/patients`
// ❌ const url = req.headers.get("x-target-url")  // SSRF risk
```

---

### CSRF Protection

OAuth2 state + PKCE (library-managed) protect the auth flow. For all other POST/PATCH/DELETE:

```ts
// Browser — add to every mutating fetch
fetch(url, {
  method: "POST",
  credentials: "include",
  headers: { "Content-Type": "application/json", "X-Requested-With": "XMLHttpRequest" },
  body: JSON.stringify(data),
})

// Edge Function — verify on every mutating handler
if (req.method !== "GET" && req.method !== "OPTIONS") {
  if (req.headers.get("x-requested-with") !== "XMLHttpRequest")
    return new Response("Forbidden", { status: 403, headers: corsHeaders })
}
```

---

### Security Headers

All custom Edge Function responses:
```ts
const securityHeaders = {
  "X-Content-Type-Options": "nosniff",
  "X-Frame-Options":        "DENY",
  "Referrer-Policy":        "strict-origin-when-cross-origin",
  "Permissions-Policy":     "camera=(), microphone=(), geolocation=()",
}
```

`index.html`:
```html
<meta http-equiv="Content-Security-Policy"
  content="default-src 'self'; connect-src 'self' https://*.supabase.co;
           script-src 'self'; style-src 'self' 'unsafe-inline';
           img-src 'self' data: https:; frame-ancestors 'none'">
```

---

### Pipeline Guardrails (Every Push to main/master)

Six automated checks — any failure blocks deployment and triggers a Codex auto-fix PR:

1. **ESLint + eslint-plugin-security** — `eval()`, dynamic regex, non-literal `require()`, child_process with variables, Unicode bidi chars. All treated as errors.
2. **Semgrep (OWASP Top 10)** — injection, broken auth, XSS, insecure deserialization, and more.
3. **Gitleaks (full git history)** — scans every past commit. A secret removed in a later commit still fails. **Never commit a secret, even temporarily — it cannot be undone without a force-push.**
4. **npm audit** — high-severity vulnerabilities fail if no Dependabot config exists (pipeline creates one).
5. **.env allowlist** — only the 8 Supabase keys above; any other key fails immediately.
6. **Build** — `npm run build` must exist and succeed with a frozen lockfile.

Branch convention: `main`/`master` → guardrails → `stable` → deploy.

---

### Prohibited Patterns — Never Do These

| Pattern | Why |
|---|---|
| `VITE_ALFADOCS_CLIENT_SECRET=...` | Exposed in browser bundle |
| `fetch("https://app.alfadocs.com/api/...")` from React | Token would need to be in the browser |
| `localStorage.setItem("token", ...)` | Stolen by XSS |
| `Access-Control-Allow-Origin: *` | Breaks credentials, insecure |
| `SameSite=Lax` or `SameSite=Strict` on session cookie | Silently breaks cross-origin auth |
| `SameSite=None` without `Secure` | Browser rejects the cookie |
| `dangerouslySetInnerHTML={{ __html: apiData }}` | XSS |
| `eval()` or `new Function(string)` with dynamic content | Code injection |
| `console.log(token)` or `console.log(session)` | Token leaks into browser logs |
| `return new Response(JSON.stringify({ error: e.message }))` | Leaks internals |
| URL built from user input in Edge Function | SSRF |
| Forwarding browser `Authorization` header to AlfaDocs | Token injection |
| `supabase.auth.*` for AlfaDocs login | Wrong auth system |
| `initAlfadocsAuth(...)` or any frontend auth client | Removed — use `createAlfadocsSupabaseAuth` in Edge Function |
| A `/callback` React page for the OAuth2 exchange | Edge Function handles it server-side |
| Rendering protected content before session check resolves | Auth bypass / content flicker |
| Unversioned Deno imports (`deno.land/x/zod/mod.ts`) | Supply chain risk |
| Missing `credentials: "include"` on fetch to Supabase | Cookie not sent, auth silently breaks |
| Missing `X-Requested-With` on mutating fetches | CSRF vulnerability |
| Missing `Access-Control-Allow-Credentials: "true"` | Browser rejects the cookie |
| Auth redirect target from URL param or user input | Open redirect vulnerability |
| Non-Supabase key in a committed `.env` file | CI pipeline env policy failure |
| Missing `npm run build` or lockfile | CI pipeline build failure |
| Secret committed to git history, even temporarily | Gitleaks finds it — irrecoverable |