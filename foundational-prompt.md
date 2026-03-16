# Alfadocs Foundational Guidelines

This document has two purposes:
1. **Prompt templates for PMs** — paste into Lovable when starting or securing a project.
2. **Persistent rules for Lovable** — always apply these rules on every edit, not just at project start.

---

## FOR PMs: Starting a New Project

> Paste the block below into Lovable as your first message, then describe your app underneath it.

---

```
I'm building a Lovable app that integrates with AlfaDocs for authentication.
Before writing any feature code, set up the full authentication and session flow first.

Architecture rules (non-negotiable):
- This is a Backend-for-Frontend (BFF) app. The React frontend never calls AlfaDocs APIs directly.
- All AlfaDocs API calls go through Supabase Edge Functions.
- The browser never holds an access token. Tokens live only in an httpOnly cookie on Supabase.

What to build first:
1. A Supabase Edge Function at supabase/functions/alfadocs-auth/ that handles:
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

3. Three routes:
   - Login page → calls auth.login() on button click
   - /callback page → calls auth.handleCallback(), then navigates to dashboard
   - App root → calls auth.refreshSession() on mount, redirects to login if false

Environment variables needed in Lovable (Secrets panel, NOT prefixed with VITE_):
   ALFADOCS_CLIENT_ID, ALFADOCS_CLIENT_SECRET, ALFADOCS_REDIRECT_URI, ALLOWED_ORIGIN

Environment variables needed in Vite (safe to expose, VITE_ prefix OK):
   VITE_ALFADOCS_CLIENT_ID, VITE_SUPABASE_URL

Do not use supabase.auth for AlfaDocs login. Do not store tokens in localStorage or sessionStorage.
Do not call AlfaDocs APIs from the browser. Always use credentials: "include" on every fetch to Supabase.

Once auth works end-to-end (login → callback → session check → logout), then build:
[DESCRIBE YOUR APP FEATURES HERE]
```

---

## FOR PMs: Retrofitting an Existing Project

> Use this if you have an existing Lovable project and want to make it follow these security rules.

---

```
I need you to audit and fix this project to follow the Alfadocs security architecture.

Audit checklist — find and fix every instance of:
1. Any fetch() call to AlfaDocs APIs (app.alfadocs.com) from the browser or React code
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

8. Any missing auth.refreshSession() call on app mount
   → Add it. Without it, the user appears logged out after every page refresh.

For each issue found, show me what you changed and why before moving on.
Once the audit is clean, we can continue with new features.
```

---

## FOR LOVABLE: Development Rules (Always Apply)

> These rules are active on every edit. You are a senior software engineer.
> Never violate these rules, even if the user's prompt does not mention security.

---

### Architecture

- The frontend (React + Vite) never calls AlfaDocs APIs directly. Ever.
- All AlfaDocs API calls happen inside Supabase Edge Functions (Deno runtime).
- The browser communicates only with Supabase Edge Functions, always with `credentials: "include"`.
- Do not use `supabase.auth` for AlfaDocs authentication — it is a different system.

```
Browser → Supabase Edge Function → AlfaDocs API
```

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
| `auth.refreshSession()` | App root on mount | Returns `true` if session is valid |
| `auth.isAuthenticated()` | Guards, conditional renders | Sync boolean, in-memory only |
| `auth.logout()` | Logout button | Clears cookie via Supabase |

Always call `refreshSession()` on mount and handle the `false` case immediately:
```tsx
useEffect(() => {
  auth.refreshSession().then(authenticated => {
    if (!authenticated) navigate('/login')
    else setLoggedIn(true)
  }).finally(() => setReady(true))
}, [])
```

---

### Environment Variables

`VITE_` prefix = baked into the browser bundle at build time = visible to anyone in DevTools.

| Variable | VITE_ prefix? | Why |
|---|---|---|
| `VITE_ALFADOCS_CLIENT_ID` | ✅ OK | Appears in the OAuth2 authorize URL anyway |
| `VITE_SUPABASE_URL` | ✅ OK | Public Supabase project URL |
| `VITE_SUPABASE_KEY` | ✅ OK | Publishable anon key, safe to expose |
| `ALFADOCS_CLIENT_SECRET` | ❌ Never | Supabase secrets only |
| Any token, API key, or credential | ❌ Never | Supabase secrets only |

Set secrets via Supabase CLI (never in `.env` files):
```bash
supabase secrets set ALFADOCS_CLIENT_ID=...
supabase secrets set ALFADOCS_CLIENT_SECRET=...
supabase secrets set ALFADOCS_REDIRECT_URI=https://yourapp.lovable.app/callback
supabase secrets set ALFADOCS_BASE_URL=https://app.alfadocs.com
supabase secrets set ALLOWED_ORIGIN=https://yourapp.lovable.app
```

Always provide `.env.example` with placeholder values. Never commit `.env`. Add `.env*` to `.gitignore`.

---

### Session Cookie

The Supabase Edge Function must set the cookie with **all** of these attributes:
```
HttpOnly    — JS cannot read it
Secure      — HTTPS only
SameSite=None — required: Lovable (lovable.app) and Supabase (supabase.co) are different origins
Path=/
Max-Age=604800
```

Never use `SameSite=Lax` or `SameSite=Strict` — they silently break cross-origin cookie sending.
`SameSite=None` requires `Secure` — browsers reject `SameSite=None` cookies without it.

---

### CORS

```ts
const corsHeaders = {
  "Access-Control-Allow-Origin": ALLOWED_ORIGIN,   // exact origin from env var — never *
  "Access-Control-Allow-Credentials": "true",
  "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
  "Access-Control-Allow-Headers": "Content-Type, X-Requested-With",
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
| React state / Context / Zustand | ❌ Never | ✅ `isAuthenticated` boolean only |
| `window.*` or globals | ❌ Never | ❌ Never |
| httpOnly cookie via Supabase | ✅ Only here | ✅ |

---

### API Proxy Pattern

Every Supabase Edge Function that calls AlfaDocs must:
1. Read the token from the session cookie — never from a request header
2. Add `Authorization: Bearer {token}` server-side
3. Never forward any `Authorization` header from the browser

```ts
// ✅ Correct
const token = getAccessTokenFromCookie(req)
if (!token) return new Response("Unauthorized", { status: 401 })
const res = await fetch(`${ALFADOCS_BASE_URL}/api/v1/...`, {
  headers: { Authorization: `Bearer ${token}` },
})

// ❌ Wrong
const res = await fetch(url, {
  headers: { Authorization: req.headers.get("Authorization") },
})
```

`ALFADOCS_BASE_URL` must come from `Deno.env.get("ALFADOCS_BASE_URL")` — never hardcoded or from user input.

---

### React Security

- Never use `dangerouslySetInnerHTML` unless sanitized with DOMPurify first
- Never interpolate user input or API data into `href`, `src`, or `style` without validation
- Never use `eval()` or `new Function(string)` with dynamic content
- Always add `rel="noopener noreferrer"` to `<a target="_blank">` links
- Never `console.log` tokens, cookies, or session data
- Treat AlfaDocs API responses as untrusted — validate shape before rendering

---

### Input Validation (Edge Functions)

Validate every request from the browser with Zod (pinned version):
```ts
import { z } from 'https://deno.land/x/zod@3.23.8/mod.ts'

const body = z.object({ code: z.string().min(1).max(512) }).parse(await req.json())
```

Never construct AlfaDocs URLs from user-supplied values (SSRF prevention):
```ts
// ✅ Safe — URL built from env var + hardcoded template
const url = `${ALFADOCS_BASE_URL}/api/v1/practices/${practiceId}/patients`

// ❌ Unsafe — SSRF risk
const url = req.headers.get("x-target-url")
```

---

### Error Handling

Never expose internal details to the browser:
```ts
// ✅ Correct
return json({ error: "Request failed" }, 500)

// ❌ Wrong
return json({ error: e.message, stack: e.stack }, 500)
```

Log full errors server-side. Never proxy AlfaDocs error responses verbatim to the browser.

---

### CSRF Protection

The OAuth2 `state` parameter (automatic via the library) protects the auth flow.

For all other POST/PATCH/DELETE calls through Supabase, add `X-Requested-With` and verify it:
```ts
// Browser
fetch(url, {
  method: "POST",
  credentials: "include",
  headers: { "Content-Type": "application/json", "X-Requested-With": "XMLHttpRequest" },
  body: JSON.stringify(data),
})

// Edge Function
if (req.method !== "GET" && req.method !== "OPTIONS") {
  if (req.headers.get("x-requested-with") !== "XMLHttpRequest") {
    return new Response("Forbidden", { status: 403 })
  }
}
```

---

### Security Headers

Include on all Edge Function responses:
```ts
"X-Content-Type-Options": "nosniff",
"X-Frame-Options": "DENY",
"Referrer-Policy": "strict-origin-when-cross-origin",
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

- `redirectUri` must always be `${window.location.origin}/callback` — never from a URL parameter
- After login/logout, only redirect to known internal routes — never to external URLs
- The Edge Function must verify `redirect_uri` matches `ALFADOCS_REDIRECT_URI` exactly

---

### Prohibited Patterns — Never Do These

| Pattern | Why |
|---|---|
| `VITE_ALFADOCS_CLIENT_SECRET=...` | Exposed in browser bundle |
| `fetch("https://app.alfadocs.com/api/...")` from React | Token would need to be in browser |
| `localStorage.setItem("token", ...)` | Stolen by XSS |
| `Access-Control-Allow-Origin: *` | Breaks credentials, insecure |
| `SameSite=Lax` or `SameSite=Strict` on the session cookie | Silently breaks cross-origin auth |
| `dangerouslySetInnerHTML={{ __html: apiData }}` | XSS |
| `console.log(token)` or `console.log(session)` | Token in browser logs |
| `return json({ error: e.message })` from Edge Function | Leaks internals |
| URL built from user input in Edge Function | SSRF |
| Forwarding browser `Authorization` header to AlfaDocs | Token injection |
| `supabase.auth.*` for AlfaDocs login | Wrong auth system |
| Skipping `refreshSession()` on app mount | User stuck in broken auth state |
| Unversioned Deno imports (`deno.land/x/zod/mod.ts`) | Supply chain risk |
| Missing `credentials: "include"` on fetch to Supabase | Cookie not sent, auth breaks |