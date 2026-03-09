You are building a web app that MUST be a **Backend-for-Frontend (BFF)** architecture, **not a pure frontend-only app**.
🚨 AUTH IS A HARD REQUIREMENT – DO NOT SKIP 🚨
- Implement **authentication and session management FIRST**, before any non-auth features.
- Authentication with AlfaDocs and secure session handling is **mandatory**. 
- Do NOT ignore, delay, or stub out the auth flow. If you are unsure, ASK the user for clarification instead of skipping it.
## Architecture
- The **BFF** (Supabase backend, Edge/Server Functions, or equivalent) is the **only** backend the frontend talks to.
- The BFF is responsible for:
  - **AlfaDocs OAuth2**: handling auth flows, token exchange, refresh, and session.
  - **Supabase services**: databases, storage, realtime, etc. — the BFF connects to these on behalf of the user; the frontend does NOT call Supabase directly for data or auth.
- The frontend (React/Next/etc.) must only call BFF endpoints (e.g. `/api/*` or Supabase functions invoked as your app’s API). It must NOT talk directly to AlfaDocs OAuth endpoints or to Supabase for sensitive/data operations.
## Authentication with AlfaDocs (CRITICAL – MUST IMPLEMENT)
- Use the official AlfaDocs helper:
  - `@alfadocs/oauth2-client` – see README at `https://github.com/alfadocs/oauth2-client`.
- Use **Authorization Code + PKCE** as in the README:
  - **Frontend (browser)**:
    - Start login and handle the redirect using:
      - `createPkcePair`
      - `buildAuthorizeUrl`
      - `parseCallback`
    - Only handle non-sensitive data (e.g. authorization code, state, PKCE verifier reference); never handle access/refresh tokens here.
  - **BFF (Supabase functions / Node)**:
    - Exchange the authorization code and handle refresh using:
      - `exchangeCodeForToken`
      - `refreshAccessToken`
    - Use `isTokenExpiredOrExpiringSoon` to decide when to refresh.
- Configuration must come from environment variables on the BFF only:
  - `ALFADOCS_CLIENT_ID`
  - `ALFADOCS_CLIENT_SECRET`
  - `ALFADOCS_REDIRECT_URI`
- Never expose secrets or tokens in frontend code.
## Sessions & Cookies (NO localStorage)
- **Do NOT use `localStorage` or `sessionStorage` for tokens or auth state.**
- BFF responsibilities:
  - Store AlfaDocs access/refresh tokens securely (e.g. in Supabase DB or a secure store).
  - Issue a short, opaque **session ID** that maps to stored tokens (and optionally to Supabase user/session).
  - Send session ID to the browser as a **secure HttpOnly cookie**:
    - `httpOnly: true`
    - `secure: true` (in production)
    - `sameSite: 'lax'` or `'strict'`
- Frontend uses only the cookie-backed session when calling BFF; the BFF uses the session to call AlfaDocs APIs and Supabase (database, etc.) with the right user context.
- Implement **login**, **logout**, and **session refresh** flows end-to-end.
## BFF as gateway for all backend access
- **AlfaDocs**: token handling and API calls happen only in the BFF.
- **Supabase (database, storage, etc.)**: the BFF connects to Supabase using server-side credentials and user context derived from the session. The frontend does not use the Supabase client directly for sensitive DB/storage access; it goes through your BFF APIs.
- Any other external APIs or services should also be called from the BFF, not from the browser.
## Security Best Practices (avoid common Lovable issues)
- **CSRF**
  - Because auth uses cookies, protect all state-changing requests (POST/PUT/PATCH/DELETE) with CSRF tokens or strict Origin/Referer checks.
- **XSS**
  - No unsafe HTML injection. Avoid `dangerouslySetInnerHTML`; if unavoidable, sanitize with a trusted library. No `eval` or dynamic script injection with untrusted data.
- **Security headers**
  - Use middleware (e.g. `helmet`) to set:
    - `Content-Security-Policy`
    - `X-Frame-Options` / `frame-ancestors`
    - `X-Content-Type-Options: nosniff`
    - `Referrer-Policy`
    - `Permissions-Policy`
- **CORS**
  - Use a strict allowlist of origins; never `origin: '*'` or reflecting arbitrary origins in production.
- **Input validation**
  - Validate and sanitize all request data on the BFF (e.g. Zod/Joi).
- **Secrets**
  - All secrets (AlfaDocs, Supabase, API keys) must come from env vars or a secret manager; provide `.env.example` without real values.
- **Errors & logging**
  - Do not expose stack traces or internal details to the client in production. Do not log tokens or sensitive PII.
## What to avoid
- Do NOT:
  - Treat this as a pure frontend app.
  - Let the frontend call AlfaDocs or Supabase directly for auth or sensitive data.
  - Store auth tokens in `localStorage`, `sessionStorage`, or non-HttpOnly cookies.
  - Expose `ALFADOCS_CLIENT_SECRET` or any token in client-side code.
  - Use wildcard CORS or skip CSRF.
  - Skip or “fake” the auth flow.
- Always:
  - Implement the **auth + session flow completely** using the BFF and `@alfadocs/oauth2-client`.
  - Use the BFF as the single gateway for AlfaDocs, Supabase (database and other services), and any other backends.
  - Prefer security and BFF architecture over frontend-only shortcuts.
