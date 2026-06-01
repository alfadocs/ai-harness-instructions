---
name: alfadocs-connected-app-oauth
description: Use when adding "Sign in with AlfaDocs" / AlfaDocs OAuth login to a Lovable app, or building a connected/marketplace app that many AlfaDocs practices will authorise (each practice logs in with their own account). Covers the OAuth2 + PKCE flow via a single Supabase Edge Function BFF.
---

# Sign in with AlfaDocs (OAuth2 + PKCE connected app)

Use this when end users authenticate with **their own AlfaDocs account** — partner apps, marketplace apps, anything many practices install. (If your app uses a single shared API key instead of per-user login, that is a different pattern — this skill does not apply.)

The whole login system is **one Supabase Edge Function**. There is no auth code in React, no OAuth library in the browser, and the Client Secret never leaves the server.

## The flow in one picture

```
React "Sign in" link
  → GET  https://<ref>.supabase.co/functions/v1/alfadocs-auth/login
  → Edge Function redirects to AlfaDocs /oauth2/authorize  (PKCE challenge + state)
  → user logs in & consents on AlfaDocs
  → AlfaDocs redirects to .../alfadocs-auth/callback?code=...&state=...
  → Edge Function (server-side) exchanges code+secret for tokens, stores them,
    sets an HttpOnly session cookie, redirects to your postLoginPath
  → React calls .../alfadocs-auth/session to confirm it's logged in
```

Tokens live server-side, keyed by an opaque `HttpOnly` cookie. The browser never sees an access token or the secret.

## Step 1 — Get an OAuth client (Client ID + Secret)

You need a Client ID + Client Secret before any code works. If you have `ROLE_ADMIN` on app.alfadocs.com, create one at **https://app.alfadocs.com/admin/marketplace-apps → Create**. Otherwise ask an AlfaDocs admin to make one for you, supplying: app name, one-line description, redirect URI (Step 3), the scopes you need (Step 4), grant types `authorization_code` + `refresh_token`, token auth `client_secret_post`.

> **The Client Secret is shown exactly once.** Copy it immediately. If you lose it, regenerate it (which invalidates the old one) and update every environment's Supabase secrets.

## Step 2 — Build the login as ONE Edge Function

Create a single file `supabase/functions/alfadocs-auth/index.ts`. Do not split callback into a second function. Do not add a `/callback` page in React.

```ts
// supabase/functions/alfadocs-auth/index.ts
import { createAlfadocsAuth } from "npm:@alfadocs/auth@0.4.1";
import { createSupabaseStorage } from "npm:@alfadocs/auth@0.4.1/supabase-bridge";

const auth = createAlfadocsAuth({
  clientId:     Deno.env.get("ALFADOCS_CLIENT_ID")!,
  clientSecret: Deno.env.get("ALFADOCS_CLIENT_SECRET")!,
  redirectUri:  Deno.env.get("ALFADOCS_REDIRECT_URI")!,   // this function's /callback URL
  appOrigin:    Deno.env.get("ALLOWED_ORIGIN")!,          // your frontend origin
  // scopes: [ ... ]  — see Step 5
  storage: createSupabaseStorage({
    supabaseUrl:    Deno.env.get("AUTH_SUPABASE_URL")!,    // project holding alfa_* tables (= SUPABASE_URL if one project)
    serviceRoleKey: Deno.env.get("AUTH_SUPABASE_KEY")!,    // SERVICE ROLE — never the anon key, never the browser
    oauthClientId:  Deno.env.get("ALFADOCS_CLIENT_ID")!,   // scopes the multi-tenant app_id on every row
  }),
  resolveProfile: async ({ accessToken, meUrl, fetchImpl }) => {
    const me = await fetchImpl(meUrl, {
      headers: { Authorization: `Bearer ${accessToken.access_token}`, Accept: "application/json" },
    }).then(r => r.json());
    // Guard: String(undefined) stores the literal "undefined" as practice_id,
    // causing every subsequent token lookup to silently return null.
    const practiceId = me.practiceId ?? me.practice_id;
    const archiveId  = me.archiveId  ?? me.archive_id;
    if (!practiceId || !archiveId)
      throw new Error(`/me missing practiceId/archiveId: ${JSON.stringify(me)}`);
    await supabaseAdmin.from("oauth_tokens").upsert({
      practice_id: String(practiceId), archive_id: String(archiveId),
      access_token: accessToken.access_token, refresh_token: accessToken.refresh_token,
      expires_at: new Date(Date.now() + (accessToken.expires_in ?? 3600) * 1000).toISOString(),
      status: "active",
    }, { onConflict: "practice_id,archive_id" });
    return { practiceId, archiveId };
  },
});

// Supabase serves this at /functions/v1/alfadocs-auth/<path>, but handleRequest expects
// bare /login, /callback, … — strip the prefix first, or no route matches.
Deno.serve((req) => {
  const u = new URL(req.url);
  const inner = u.pathname.replace(/^\/functions\/v1\/alfadocs-auth/, "") || "/";
  // new Request() requires an absolute URL — reconstruct from base, not bare path
  return auth.handleRequest(new Request(new URL(inner + u.search, req.url).href, req));
});
```

**Two one-time setup steps** the bridge needs: (1) apply the `@alfadocs/auth` migrations so the `alfa_users` / `alfa_sessions` tables exist — `supabase db push`, or paste the migration SQL into the Supabase SQL editor (they are **not** auto-created); (2) set `verify_jwt = false` for this function in `supabase/config.toml`, or the gateway demands a Supabase JWT before the AlfaDocs cookie exists.

The library handles every route behind this one function:

| Method | Path | Does |
|---|---|---|
| `GET` | `/login` | Redirect to AlfaDocs `/oauth2/authorize` with PKCE + `state` |
| `GET` | `/callback` | Validate `state`, exchange code for tokens, store them, set cookie, redirect to `postLoginPath` |
| `GET` | `/session` | `{ authenticated: true, user }` or `{ authenticated: false }` |
| `POST` | `/logout` | Verify `Origin`, delete session, clear cookie |
| `OPTIONS` | any | CORS preflight scoped to `appOrigin` |

## Step 3 — Set the Supabase secrets

Set these in **Supabase → Project → Edge Functions → Secrets** (or `supabase secrets set`). Set them on **every** environment (preview and production each have their own Supabase project / `<project-ref>`).

```
ALFADOCS_CLIENT_ID        # from Step 1
ALFADOCS_CLIENT_SECRET    # from Step 1 — secret, server-only
ALFADOCS_REDIRECT_URI     # https://<project-ref>.supabase.co/functions/v1/alfadocs-auth/callback
ALLOWED_ORIGIN            # your frontend URL, e.g. https://my-app.lovable.app  (NEVER "*")
AUTH_SUPABASE_URL         # project holding alfa_users/alfa_sessions (= your SUPABASE_URL if one project)
AUTH_SUPABASE_KEY         # that project's SERVICE ROLE key — server-only, never the anon key
```

**Never** put any of these in `.env`, in a `VITE_*` variable, or in committed files. Secrets in git are unrecoverable. The frontend only ever needs `VITE_SUPABASE_URL` (the function base URL) — not the client ID or secret.

## Step 4 — The Callback URL must match exactly

The Callback URL you register with AlfaDocs is **this Edge Function's `/callback` route**, and it is the same value as `ALFADOCS_REDIRECT_URI`:

```
https://<project-ref>.supabase.co/functions/v1/alfadocs-auth/callback
```

AlfaDocs enforces an **exact-match** on the redirect URI — scheme, host, path, trailing slash, everything. One character off → the callback fails with `400`. Register one per environment (preview project-ref and production project-ref are different). Add the production one after your first production deploy, once that Supabase project exists.

## Step 5 — Request only the scopes you need

Scopes are a single **space-separated** `scope` value. The user sees them on the consent screen, so over-asking loses trust — list only what the app reads or writes. The full catalogue and example scope sets are in [scopes.md](./scopes.md).

## Step 6 — The React side (two routes, no auth library)

```tsx
// src/components/LoginPage.tsx
const EDGE = import.meta.env.VITE_SUPABASE_URL + "/functions/v1/alfadocs-auth";
export const LoginPage = () => <a href={`${EDGE}/login`}>Sign in with AlfaDocs</a>;
```

```tsx
// src/components/ProtectedRoute.tsx
import { useEffect, useState } from "react";
import { Navigate, Outlet } from "react-router-dom";

const EDGE = import.meta.env.VITE_SUPABASE_URL + "/functions/v1/alfadocs-auth";

export const ProtectedRoute = () => {
  const [state, setState] = useState<"loading" | "authed" | "anon">("loading");
  useEffect(() => {
    fetch(`${EDGE}/session`, { credentials: "include" })   // credentials: "include" is required
      .then(r => r.json())
      .then(({ authenticated }) => setState(authenticated ? "authed" : "anon"))
      .catch(() => setState("anon"));
  }, []);
  if (state === "loading") return <Spinner />;
  if (state === "anon") return <Navigate to="/login" replace />;
  return <Outlet />;
};
```

- **No `/callback` route in React** — the callback is server-side.
- **No Supabase Auth `signIn`** for AlfaDocs login — wrong system; use the Edge Function `/login`.
- Always send `credentials: "include"` so the session cookie travels.

## Step 7 — Get practiceId + archiveId, then call the API

Every AlfaDocs API call needs `practiceId` and `archiveId`. Get them once after login by calling `/me`, then reuse them. **Make these calls from a server-side Edge Function**, not the browser — the browser must never hold the access token.

```ts
// inside a custom Edge Function, after looking up the session's access token
const headers = {
  "Authorization": `Bearer ${accessToken}`,   // OAuth → Bearer only (X-Api-Key is the separate API-key scheme)
  "Accept": "application/json",
};

const me = await fetch("https://app.alfadocs.com/api/v1/me", { headers }).then(r => r.json());
const { practiceId, archiveId } = me;   // required for every other endpoint

// e.g. list patients
await fetch(
  `https://app.alfadocs.com/api/v1/practices/${practiceId}/archives/${archiveId}/patients`,
  { headers },
);
```

- Base URL: `https://app.alfadocs.com/api/v1`
- API reference: `https://app.alfadocs.com/api.html` — check it for the exact path/shape of every endpoint.

## Custom Edge Functions that call AlfaDocs

To proxy any AlfaDocs call, write another Edge Function that:

1. Reads the session cookie and looks up the access token server-side.
   **If the session is expired or missing, return a 401 immediately** — the frontend
   must handle this by redirecting to `/login`, not showing a generic error card:
   ```ts
   // bff.ts / Wallboard fetch
   if (response.status === 401) { window.location.href = loginUrl(window.location.origin); return; }
   ```
2. **Checks if the token is expired and refreshes it if so** (short-lived tokens will expire during long sessions — a wallboard or polling app will 401 after ~1h without refresh). Use the concurrency-safe row-lock pattern from the `alfadocs-oauth-token-refresh` skill. Store `expires_at` as an **absolute timestamp**, not a delta, so you can compare `expires_at < now()` directly.
3. Calls AlfaDocs with the headers above (OAuth → `Authorization: Bearer` + `Accept`; **not** `X-Api-Key` — that's the separate API-key scheme).
4. Returns only what the frontend needs — never the raw token.

Every non-`GET` fetch from React must send `X-Requested-With: XMLHttpRequest`, and your function must reject mutating requests that lack it:

```ts
if (req.method !== "GET" && req.headers.get("x-requested-with") !== "XMLHttpRequest") {
  return new Response("Forbidden", { status: 403 });
}
```

## UI / design system

Use **@alfadocs/ui-kit** for all UI (public npm: `npm install @alfadocs/ui-kit`, React 18 or 19). Import tokens once at the app root, wrap the tree in `<ThemeRoot theme="light">`, route all visible strings through `useTranslation()`, and reference `var(--*)` tokens instead of hex/rgb. Browse components at **storybook.alfadocs.com**. If the component you want isn't there, stop and ask rather than pulling in another UI library.

**Don't hand-roll the login screen or the app chrome** — the kit ships both (kit **0.36+**):

- **`ConnectWithAlfadocs`** — the pre-auth "Connect with AlfaDocs" screen. Wire `onConnect` to start the flow below; drive `status` (`idle`/`connecting`/`error`) + `error` from the result.
- **`MarketplaceAppShell`** — the authed app frame (header product lockup, sidebar nav, account menu). Pass `user` + `onSignOut`.

Both are **auth-decoupled** — the kit never touches OAuth or secrets. `@alfadocs/auth/react`'s `useAlfadocsAuth` hook (now on npm) is a handy client-side source for `user`/auth state, with tokens still held server-side by the BFF. Import from `@alfadocs/ui-kit/patterns/marketplace-app-shell`.

## Things to never do

| Anti-pattern | Use instead |
|---|---|
| `fetch("https://app.alfadocs.com/api/...")` from React | An Edge Function proxy |
| Client Secret / access token in `VITE_*` or `.env` | `Deno.env.get(...)` inside the Edge Function |
| `localStorage.setItem("token", ...)` | Server-side session lookup via the cookie |
| A `/callback` page in React | The Edge Function `/callback` route |
| A second `*-callback` Edge Function | One `alfadocs-auth` function does login + callback |
| `Access-Control-Allow-Origin: "*"` | `ALLOWED_ORIGIN` exactly — and **reject non-matching origins with 403**, not just omit the header: `const allow = origin === APP_ORIGIN ? APP_ORIGIN : null; if (!allow) return new Response("Forbidden", { status: 403 });` |
| Redirect URI that doesn't byte-match the registered one | Register the exact `/callback` URL |
| `supabase.auth.signIn(...)` for AlfaDocs login | Edge Function `/login` |
| `console.log(token)` | Don't log tokens |
| Supabase browser client with `persistSession: true` / `localStorage` | Set `persistSession: false, storage: undefined` — the app uses cookie-BFF auth, not Supabase Auth. Supabase Auth session storage is an unused XSS vector here. |
| Token refresh skipped / `expires_in` stored as a delta | Store `expires_at` as an absolute ISO timestamp (`new Date(Date.now() + expires_in * 1000).toISOString()`) and check `expires_at < now()` before every API call. Long-running sessions (polling, TV wallboards) will 401 after ~1h otherwise. |
| 401 from Edge Function shows a generic error card | Redirect to `/login` — the session has expired and retrying won't help: `if (res.status === 401) { window.location.href = loginUrl(window.location.origin); return; }` |
| `String(me.practiceId)` without a null-check | Guard first: `const pid = me.practiceId ?? me.practice_id; if (!pid) throw new Error('missing practiceId');` — `String(undefined)` stores the literal `"undefined"` and every subsequent token lookup silently returns nothing. |
