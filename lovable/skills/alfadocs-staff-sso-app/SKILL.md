---
name: alfadocs-staff-sso-app
description: Use when building an INTERNAL tool for AlfaDocs staff/employees (not sold to customers, not a single external practice) — e.g. an internal dashboard, ops or workplan tool — where employees sign in with their @alfadocs.com Google Workspace account, the app stores its own data in Supabase, and it OPTIONALLY reads AlfaDocs practice data server-side via an API key. NOT a customer-facing or multi-practice marketplace product (that is alfadocs-connected-app-oauth).
---

# Build an internal AlfaDocs staff app (staff SSO + own data)

This is for an **internal AlfaDocs tool** — built for an **internal AlfaDocs audience** (employees), not sold to customers. Think an internal dashboard or an ops/workplan tool. Three things define it:

1. **Staff log in** with their `@alfadocs.com` Google Workspace account (Supabase Auth).
2. The app **owns its data** in Supabase (its own tables, its own schema).
3. It **optionally** reads AlfaDocs practice data **server-side** via an API key.

### Which archetype am I?

| If… | Use |
|---|---|
| Internal staff tool, employees sign in, own data (this) | **`alfadocs-staff-sso-app`** |
| Customer product, many practices each "Sign in with AlfaDocs" | `alfadocs-connected-app-oauth` |
| One external practice, no user login, API key only | `alfadocs-connected-app-api-key` |

If your users are customers, stop — this is the wrong skill.

## Architecture

```
React (DS frame)
  → Supabase Auth      (Google Workspace @alfadocs.com)   → alfadocs-google-workspace-auth
  → Supabase DB        (your own tables, RLS by auth.uid()/role)
  → Supabase Edge Fn   (OPTIONAL: alfadocs-api proxy holding ALFADOCS_API_KEY)
```

- **Auth** is staff SSO. Follow **`alfadocs-google-workspace-auth`** for the Google provider, the `@alfadocs.com` domain lock, the route guard, and RLS on the email/role claim.
- **Data** lives in your Supabase project. This app is the source of truth for its own data.
- **AlfaDocs reads** are optional and always server-side (next section).

## Tenancy: this is NOT the practiceId model

The other two archetypes are multi-tenant by `practiceId` + `archiveId`. **A staff app is not.** There is one logical tenant — AlfaDocs itself. So:

- Scope your **own** tables by **`auth.uid()` and/or a role**, never by `practiceId`.
- Do **not** copy the "every row carries practiceId / RLS by practiceId" rule from the API-key or OAuth skills — it doesn't apply here.
- If you mirror AlfaDocs data, it's AlfaDocs's own single archive, not per-customer tenants.

## Optional: reading AlfaDocs data (server-side only)

Only if the tool needs live AlfaDocs data. The API key gives **full read/write** to that practice's data — treat it like a password and keep it in an Edge Function secret, never the browser (the rules in `alfadocs-connected-app-api-key` apply).

The one difference from that skill: **this app has a logged-in user**, so the proxy must **verify the staff Supabase session first**, then use the API key for the AlfaDocs hop. Keep `verify_jwt = true` (the default) for this function.

```ts
// supabase/functions/alfadocs-api/index.ts (sketch)
Deno.serve(async (req) => {
  // 1. Require a valid staff Supabase session (verify_jwt=true already enforces a JWT;
  //    additionally confirm the @alfadocs.com domain on the verified user).
  // 2. Read ALFADOCS_API_KEY from Deno.env — never return it.
  // 3. GET https://app.alfadocs.com/api/v1/me  → practiceId + archiveId (cache per cold start).
  // 4. Forward the requested path with headers:
  //      X-Api-Key: <ALFADOCS_API_KEY>
  //      Accept: application/json
  // 5. Return AlfaDocs JSON + status to the frontend.
});
```

Always verify endpoints at **https://app.alfadocs.com/api.html** before mapping — don't guess paths or response shapes. (Note: many API routes are owner-gated by the platform whitelist; if a read 401/403s, it may not be reachable by API key.)

## UI

Build all UI with **`@alfadocs/ui-kit`** — see `alfadocs-design-system` for the full wiring. For this archetype specifically:

- App chrome: **`MarketplaceAppShell`** (`variant="standalone"`) with the signed-in staff `user`. There is **no** `ConnectWithAlfadocs` / "Sign in with AlfaDocs" screen — login is the Google `SocialSignInButton` (`alfadocs-google-workspace-auth`).
- The shell renders a "{productName} by Alfadocs" lockup — fine for an internal AlfaDocs tool; leave it.

## Secrets

| Secret | Where | Needed |
|---|---|---|
| Google client ID/secret | Supabase **Auth provider** config | always (staff login) |
| `ALFADOCS_API_KEY` | Supabase **Edge Function** secret | only if reading AlfaDocs data |
| `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY` | frontend env | always (these are public/publishable) |

Nothing else belongs in the frontend or in `.env` committed to git.

## Don't

- Don't add `@alfadocs/auth`, OAuth/PKCE, a per-practice token store, or a marketplace listing — those are the customer-product archetype.
- Don't put `ALFADOCS_API_KEY` (or any secret) in the browser or a `VITE_*` var.
- Don't scope your data by `practiceId`; scope by `auth.uid()`/role.
- Don't weaken the domain lock or RLS to "make it work".

## Quick checklist

- [ ] Staff sign in with `@alfadocs.com` Google Workspace (per `alfadocs-google-workspace-auth`).
- [ ] Own Supabase tables, RLS deny-by-default, scoped by `auth.uid()`/role (not `practiceId`).
- [ ] AlfaDocs reads (if any) go through an Edge Function that verifies the staff session, holds `ALFADOCS_API_KEY` server-side, calls `/me` first, and never returns the key.
- [ ] Endpoints verified at https://app.alfadocs.com/api.html.
- [ ] UI built with `@alfadocs/ui-kit`; chrome = `MarketplaceAppShell` standalone; no `ConnectWithAlfadocs`.
- [ ] No `@alfadocs/auth`, no marketplace OAuth, no secrets in the frontend.
