---
name: alfadocs-connected-app-api-key
description: Use when building a private or internal tool for a single AlfaDocs practice to read or write its own data with an API key (no user login) — a dashboard, report, or automation over patients, appointments, or other practice data, calling https://app.alfadocs.com/api/v1 server-to-server.
---

# Build an AlfaDocs API-key app (single practice, no login)

This is for a **single practice using its own AlfaDocs data** in a private/internal tool — a report, dashboard, or automation. There is **no user login**: the app authenticates to AlfaDocs with **one API key** that belongs to that one practice.

If your app instead needs *end users to log in with their own AlfaDocs account* (a multi-practice or customer-facing product), this is the wrong skill — that needs OAuth 2.0, not an API key.

## The golden rule: the key never touches the frontend

The API key gives **full read/write access to the practice's data**. Treat it like a password.

- Store it as a **Supabase Edge Function secret** named `ALFADOCS_API_KEY`.
- **Never** put it in frontend code, in a `VITE_*` variable, in a `.env` committed to git, or in any value the browser can read.
- All AlfaDocs API calls go **through a Supabase Edge Function** (a proxy / BFF). The browser calls *your* Edge Function; the Edge Function adds the key and calls AlfaDocs.

If the browser can see the key, the design is wrong — stop and move the call into the Edge Function.

### Setting the secret

In Lovable, open the Supabase integration and add an Edge Function secret:

```
Name:  ALFADOCS_API_KEY
Value: <the key from the practice's "Chiavi" / Keys screen>
```

Read it inside the function only: `Deno.env.get("ALFADOCS_API_KEY")`.

## How the AlfaDocs API works

- **Base URL:** `https://app.alfadocs.com/api/v1`
- **Headers on every call:**
  ```
  X-Api-Key: <ALFADOCS_API_KEY>
  Accept: application/json
  ```
- **Call `/me` first.** `GET https://app.alfadocs.com/api/v1/me` returns the `practiceId` and `archiveId` for the key. **Both are required for every other call.** Fetch them once, then reuse them.
- **Everything else is scoped under those IDs**, e.g. patients:
  ```
  GET https://app.alfadocs.com/api/v1/practices/{practiceId}/archives/{archiveId}/patients
  ```
- **Full endpoint reference:** https://app.alfadocs.com/api.html — always check it for the exact path, method, and body shape before building a call. Don't guess endpoints.

## The Edge Function proxy

One Edge Function is enough. It reads the secret, calls `/me` to get `practiceId` + `archiveId` (cache them in memory per cold start), then forwards the requested AlfaDocs path. See the [reference Edge Function](./reference-edge-function.md) for a complete, copy-pasteable example.

Minimum it must do:

1. Read `ALFADOCS_API_KEY` from `Deno.env.get`.
2. Resolve `practiceId` + `archiveId` from `GET /me`.
3. Forward to the AlfaDocs endpoint with the two required headers.
4. Return AlfaDocs's JSON (and status) back to the frontend. Never return the key itself.

## Data isolation by practiceId

There is exactly one practice here, but still build as if there could be more:

- Treat `practiceId` (and `archiveId`) as the **tenant**. Every stored row that mirrors AlfaDocs data should carry the `practiceId` it belongs to.
- If you persist anything in Supabase tables, scope it by `practiceId` and protect it with **Row Level Security** so data from one practice can never be read, created, updated, or deleted under another.
- Never let a frontend request choose an arbitrary `practiceId`/`archiveId` — the Edge Function decides them from `/me`, so the browser can't reach another practice's data.

## Webhooks (only if AlfaDocs calls your app)

If you register a webhook so AlfaDocs notifies your app of changes:

- Bodies arrive as `application/x-www-form-urlencoded` with **indexed bracket notation** (e.g. `items[0][id]`). Parse accordingly.
- Look up the practice from the **`archiveId`** in the payload.
- **Always respond `HTTP 200`** — even for events you ignore. Do not return 202.

## UI: use @alfadocs/ui-kit

Build all UI with the AlfaDocs design system (public npm package):

- Install: `npm install @alfadocs/ui-kit` (React 18 or 19).
- Import the tokens once at the app root: `import '@alfadocs/ui-kit/tokens';`.
- Wrap the app in `<ThemeRoot theme="light">`.
- Reference colours, spacing, radii via the `var(--*)` tokens — never raw hex/rgb/hsl.
- Put user-visible strings through `useTranslation()` under the `app` namespace (kit strings use `ui.*`).
- Don't reach for Material UI, Bootstrap, shadcn, Chakra, Mantine, or hand-rolled CSS. If a component you need isn't in the kit, stop and ask rather than pulling in an external library.
- Browse available components at **https://storybook.alfadocs.cloud**.

## Getting the API key

The key comes from the practice's AlfaDocs **Chiavi (Keys)** screen under **Impostazioni → Studio** (Settings → Practice).

- The Keys screen only shows if the user has the **"Visualizza e modifica le Chiavi API"** permission (an Owner enables it per user) and the practice is not a German practice.
- **Creating** a key requires being the practice **Owner**. If you're not, ask the Owner to create one and share it securely, or ask the AlfaDocs team.
- Treat the key like a password — once you have it, drop it straight into the `ALFADOCS_API_KEY` Edge Function secret and nowhere else.

## Quick checklist

- [ ] Key stored as Supabase secret `ALFADOCS_API_KEY`, not in frontend/.env.
- [ ] All AlfaDocs calls go through the Edge Function with `X-Api-Key` + `Accept: application/json`.
- [ ] `/me` called first; `practiceId` + `archiveId` reused for every other call.
- [ ] Endpoints verified against https://app.alfadocs.com/api.html.
- [ ] Stored data scoped + RLS-protected by `practiceId`.
- [ ] Webhooks (if any) parse bracketed form data, look up by `archiveId`, return 200.
- [ ] UI built with `@alfadocs/ui-kit`.
