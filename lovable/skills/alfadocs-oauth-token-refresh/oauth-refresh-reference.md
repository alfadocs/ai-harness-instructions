# AlfaDocs OAuth token refresh — full reference

Detailed companion to the `alfadocs-oauth-token-refresh` skill. Read the SKILL.md first for the non-negotiables and the lock pattern; this file fills in the complete flow, the cron, and the edge cases.

All tokens live in a Supabase `oauth_tokens` table, read/written only from Supabase Edge Functions using the service-role client (`supabaseAdmin`). The OAuth client secret and any API secrets live in Supabase Edge Function secrets — never in the frontend, never in a committed `.env`.

Build `supabaseAdmin` from `AUTH_SUPABASE_URL || SUPABASE_URL` and `AUTH_SUPABASE_KEY || SUPABASE_SERVICE_ROLE_KEY` — Lovable Cloud auto-injects `SUPABASE_URL` / `SUPABASE_SERVICE_ROLE_KEY`, while a standalone Supabase project uses the `AUTH_*` names, so reading both makes the token store work in either setup.

```ts
const supabaseAdmin = createClient(
  Deno.env.get("AUTH_SUPABASE_URL") ?? Deno.env.get("SUPABASE_URL")!,
  Deno.env.get("AUTH_SUPABASE_KEY") ?? Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!,
);
```

## Complete token-refresh flow (token manager)

A single `getValidToken(supabaseAdmin, { practiceId, archiveId, forceRefresh })` function should run:

1. **Read** the token row for this `practice_id` + `archive_id`.
2. If `status === "revoked"` → throw `needsReauth`.
3. If `forceRefresh` is NOT set and `expires_at > now() + 5 min` → return `access_token` (outcome: `already_fresh`).
4. If there is no `refresh_token` → throw `needsReauth`.
5. **Acquire the concurrency lock** (atomic conditional update on `refresh_locked_at`, scoped to this practice/archive):
   - If the lock is held by another process: wait ~3s, re-read the row, and **validate `expires_at`**.
     - Fresh now → return it (`already_fresh`).
     - Still stale after 2 attempts → throw a transient error. Never return a known-stale token.
6. Call AlfaDocs `POST /oauth2/token` with the `refresh_token` grant.
   - 401/400/403 → mark `status: "revoked"`, release lock, throw `needsReauth`.
   - 429/5xx/network → retry with exponential backoff (max 3 attempts).
7. **Persist** the new `access_token` and rotated `refresh_token` (using `??`, never `||`) plus the new `expires_at` in a single DB write.
8. **Release the lock** in a `finally` block (`refresh_locked_at: null`).
9. Return the new `access_token`.

## Refresh-token grant request

Exchange the refresh token at the AlfaDocs OAuth2 token endpoint. The client secret comes from Supabase Edge Function secrets:

```ts
const res = await fetch("https://app.alfadocs.com/oauth2/token", {
  method: "POST",
  headers: { "Content-Type": "application/x-www-form-urlencoded" },
  body: new URLSearchParams({
    grant_type: "refresh_token",
    refresh_token: tokenRow.refresh_token,
    client_id: Deno.env.get("ALFADOCS_CLIENT_ID")!,
    client_secret: Deno.env.get("ALFADOCS_CLIENT_SECRET")!,
  }),
});
```

On success, persist with nullish coalescing so an empty rotated field never falls back to the revoked token:

```ts
await supabaseAdmin
  .from("oauth_tokens")
  .update({
    access_token:  data.access_token,
    refresh_token: data.refresh_token ?? tokenRow.refresh_token, // ?? not ||
    expires_at:    new Date(Date.now() + data.expires_in * 1000).toISOString(),
    status:        "active",
  })
  .eq("id", tokenRow.id);
```

If the DB write fails *after* a successful HTTP refresh, log the raw response and handle the error explicitly — the new refresh token must not be lost while the old one is already revoked.

## Retry with exponential backoff

Transient errors (5xx, 429, network timeouts) retry. Revoked tokens (401/400/403) do not.

```ts
async function refreshWithRetry(supabaseAdmin, tokenRow, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await refreshOAuthToken(supabaseAdmin, tokenRow);
    } catch (err) {
      if (err.needsReauth) {
        await markTokenRevoked(supabaseAdmin, tokenRow.id);
        throw err; // permanently dead — don't retry
      }
      if (attempt < maxRetries - 1) {
        await new Promise(r => setTimeout(r, 1000 * 2 ** attempt));
      } else {
        throw err;
      }
    }
  }
}
```

## Force refresh and upstream 401 retry

The token manager's `forceRefresh` option bypasses the expiry-buffer check (step 3) and goes straight to lock + refresh. Use it when an upstream AlfaDocs API call returns 401 even though the token looked valid.

Upstream 401 handling in a caller (e.g. an `alfadocs-get-patient` Edge Function):

1. On the first 401 from the AlfaDocs API, call `getValidToken({ forceRefresh: true })`.
2. Retry the API call **once** with the fresh token.
3. Retry succeeds → return the result normally.
4. Retry fails:
   - token manager threw `needsReauth: true` → return `{ needs_reauth: true }` (genuine revocation).
   - otherwise → return HTTP 502 with `{ needs_reauth: false }` (transient issue).

This stops a single stale access token from triggering a false manual re-auth prompt.

## Rate limiting

AlfaDocs enforces ~10 req/s. When refreshing multiple practices' tokens in a loop, add ~150 ms between iterations (~6.6 req/s, safely under the limit):

```ts
await new Promise(r => setTimeout(r, 150));
```

## Proactive refresh (cron): must be non-blocking

Schedule a background refresh of near-expiry tokens with `pg_cron` (e.g. every 30 minutes) calling a Supabase Edge Function.

**The cron function must be non-blocking.** Return HTTP 200 immediately, then run the refresh loop in the background with `EdgeRuntime.waitUntil()`. Otherwise the cron caller's HTTP timeout (~5s) can kill the refresh mid-flight — exactly when a rotation is in progress, which is the worst moment to be interrupted.

```ts
const backgroundWork = (async () => {
  // for each active token nearing expiry:
  //   getValidToken(...), then await new Promise(r => setTimeout(r, 150));
})();
EdgeRuntime.waitUntil(backgroundWork);
return new Response(
  JSON.stringify({ status: "processing_in_background" }),
  { status: 200 },
);
```

In the cron query, **exclude revoked tokens**: `.neq("status", "revoked")` — otherwise the cron endlessly retries dead connections.

Log an outcome per token at the end of the run:

- `refreshed` — token was near-expiry and successfully refreshed.
- `already_fresh` — token was still valid, no action taken.
- `locked` — another process was already refreshing this token.
- `needs_reauth` — token is permanently revoked, user must re-authenticate.
- `failed` — transient error after all retries exhausted.

## OAuth callback: reset state on re-authentication

When a user successfully reconnects, the token upsert must **explicitly reset all state** so a previously revoked or stale-locked connection comes back clean:

```ts
{
  access_token:      newAccessToken,
  refresh_token:     newRefreshToken,
  expires_at:        expiresAt,
  status:            "active",   // ← clear any "revoked" state
  refresh_locked_at: null        // ← clear any abandoned lock
}
```

## Webhook / caller error classification

When webhook processing or any caller invokes an AlfaDocs-touching function, the message shown to the user must distinguish genuine revocation from a transient issue:

- `needs_reauth: true` → "OAuth token revoked. Re-authenticate in Configuration → Connections."
- `needs_reauth: false` or absent → "Token refresh in progress or temporary auth issue, retrying shortly."

**Never** map a raw HTTP 401/403 unconditionally to a re-auth prompt. The upstream function has already attempted force-refresh + retry — trust its `needs_reauth` signal.

## Multi-practice safety

Every token operation is scoped by `practice_id` + `archive_id`:

- Lock acquisition, refresh, persist, and cron iteration all filter on both.
- A token issued for practice A must **never** be used to call the API for practice B — even within the same Supabase project. The lock is per-row, so concurrent refreshes for different practices are independent and safe; only same-practice concurrency needs the mutex.
- `practice_id` and `archive_id` come from `GET /me` on `https://app.alfadocs.com/api/v1` (docs: https://app.alfadocs.com/api.html) and are required on every other call.

## Common pitfalls

| Pitfall                                          | Consequence                                       | Prevention                                                    |
| ------------------------------------------------ | ------------------------------------------------- | ------------------------------------------------------------- |
| `\|\|` instead of `??` for the refresh token      | Empty string falls back to the revoked token      | Always use `??`                                               |
| No concurrency lock                              | Two processes revoke each other's tokens          | `refresh_locked_at` row mutex                                 |
| Retrying 401/400/403                             | Wastes requests on permanently dead tokens        | Check the `needsReauth` flag                                  |
| No rate limiting in batch refresh                | 429 errors from AlfaDocs                          | 150 ms delay between iterations                               |
| Not marking revoked tokens                       | Cron endlessly retries dead tokens                | Set `status: "revoked"` on 401/400/403; `.neq("status",...)`  |
| DB write fails after successful HTTP refresh     | New refresh token lost, old one already revoked   | Log raw response, handle DB errors explicitly                 |
| Returning a stale token while the lock is held   | Caller gets 401 → false re-auth prompt            | Validate `expires_at` after the lock-wait                     |
| Blocking cron handler                            | 5s HTTP timeout kills refresh mid-flight          | `EdgeRuntime.waitUntil()`, return 200 immediately             |
| Not resetting state on re-auth                   | Revoked/locked status persists after reconnect    | Reset `status: "active"`, `refresh_locked_at: null`           |
| Unconditional 401 → re-auth in callers/webhooks  | False re-auth prompts on transient issues         | Trust `needs_reauth` from the upstream response               |
| Tokens in the browser or a committed `.env`      | Credential leak / token theft                     | Tokens server-side only; secrets in Supabase Edge secrets     |
| Using one practice's token for another           | Wrong-practice data access / auth failures        | Scope every query by `practice_id` + `archive_id`             |
