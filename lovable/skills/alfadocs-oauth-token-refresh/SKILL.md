---
name: alfadocs-oauth-token-refresh
description: Use when AlfaDocs OAuth access tokens expire, when you need to refresh a token with the refresh_token grant, or when a call to the AlfaDocs API (https://app.alfadocs.com/api/v1) returns 401/403 — covers the concurrency-safe row-lock refresh pattern, refresh-token rotation, multi-practice token isolation, and keeping all tokens server-side in Supabase.
---

# Refreshing AlfaDocs OAuth tokens safely

AlfaDocs access tokens are short-lived. You renew them with the OAuth2 `refresh_token` grant. AlfaDocs uses **refresh-token rotation**: every successful refresh revokes the old refresh token and issues a brand-new one. A single mishandled refresh — a double-refresh from two concurrent requests, or a fallback to an already-revoked token — permanently kills the connection and forces the user to re-authenticate.

This skill is the safe pattern. Follow it exactly.

## Non-negotiables (read first)

1. **Tokens live server-side only.** Store `access_token` and `refresh_token` in a Supabase table, read/written exclusively from Edge Functions using the service-role key. Never send a refresh token to the browser, never put any token in `.env` committed to git, never expose the OAuth client secret to the frontend. Secrets go in Supabase Edge Function secrets.
2. **One token per practice/archive.** Each AlfaDocs practice (and its archive) has its own token. **Never** use one practice's token to call the API for another practice. Scope every read and write by `practice_id` + `archive_id`.
3. **Persist the rotated refresh token with `??`, never `||`.** Rotation may legitimately return an empty-string field; `||` would silently fall back to the now-revoked old token and brick the connection.

```ts
// ❌ empty string falls back to the already-revoked old token
refresh_token: data.refresh_token || row.refresh_token
// ✅ only null/undefined fall back
refresh_token: data.refresh_token ?? row.refresh_token
```

## Database shape

Table `oauth_tokens`, one row per practice/archive connection:

| Column              | Type          | Notes                                              |
| ------------------- | ------------- | -------------------------------------------------- |
| `practice_id`       | text/int      | from `/me` — scopes the token                      |
| `archive_id`        | text/int      | from `/me` — scopes the token                      |
| `access_token`      | text          | short-lived bearer used for API calls              |
| `refresh_token`     | text          | rotated on every refresh                           |
| `expires_at`        | timestamptz   | when the access token dies                         |
| `status`            | text          | default `active`; set `revoked` when dead          |
| `refresh_locked_at` | timestamptz   | concurrency mutex, default `null`                  |

`practice_id` + `archive_id` come from calling `GET /me` on `https://app.alfadocs.com/api/v1` while authenticated (see https://app.alfadocs.com/api.html). They are required for every other API call — store them with the token.

## The concurrency lock — why and how

If two requests notice the token is expired at the same time, they both refresh, both rotate, and the second rotation invalidates the first's new refresh token. The connection dies. Prevent this with a **row-level lock**: an atomic conditional update on `refresh_locked_at`.

```ts
// Acquire the lock: succeeds only if no one holds it (or the lock is stale > 2 min)
const staleCutoff = new Date(Date.now() - 2 * 60 * 1000).toISOString();
const { data: lockedRow } = await supabaseAdmin
  .from("oauth_tokens")
  .update({ refresh_locked_at: new Date().toISOString() })
  .eq("practice_id", practiceId)
  .eq("archive_id", archiveId)                       // ← scope to THIS practice
  .or(`refresh_locked_at.is.null,refresh_locked_at.lt.${staleCutoff}`)
  .select("*")
  .single();
```

- **Lock acquired** (`lockedRow` returned): you own the refresh. Call AlfaDocs, persist, then release in `finally`.
- **Lock NOT acquired**: another process is refreshing. Wait ~3s, re-read the row, and **only return the token if `expires_at` is actually in the future**. After 2 failed re-checks, throw a transient error — never hand back a known-stale token.
- **Stale lock** (`refresh_locked_at` older than 2 minutes): treat as abandoned; the `.or(... .lt.)` clause lets you overwrite it.
- **Never** call `/oauth2/token` without holding the lock.

Always release the lock:

```ts
try {
  accessToken = await refreshOAuthToken(supabaseAdmin, lockedRow);
} finally {
  await supabaseAdmin
    .from("oauth_tokens")
    .update({ refresh_locked_at: null })
    .eq("id", lockedRow.id);
}
```

## Error classification — don't retry dead tokens

| Upstream status | Meaning                         | Recoverable?          |
| --------------- | ------------------------------- | --------------------- |
| 401             | Revoked / invalid refresh token | No — needs re-auth    |
| 400             | Expired auth code               | No — needs re-auth    |
| 403             | Forbidden                       | No — needs re-auth    |
| 429             | Rate limited                    | Yes, backoff          |
| 5xx             | Server error                    | Yes, retry            |

On 401/400/403, mark the row revoked, clear the lock, and signal `needsReauth` — do not retry:

```ts
if ([400, 401, 403].includes(res.status)) {
  await supabaseAdmin
    .from("oauth_tokens")
    .update({ status: "revoked", refresh_locked_at: null })
    .eq("id", tokenRow.id);
  const err = new Error(`Token revoked (${res.status}). Re-authentication required.`);
  (err as any).needsReauth = true;
  throw err;
}
```

On 429/5xx/network errors, retry with exponential backoff (max ~3 attempts, `1000 * 2 ** attempt` ms). Skip retries entirely when `needsReauth` is set.

## Handling a 401 from the AlfaDocs API (the common case)

When a normal API call (e.g. fetching a patient) returns **401 despite a token that looked valid**, the access token expired between your check and the call. Do this — never prompt the user for re-auth on the first 401:

1. Call your token manager with `{ forceRefresh: true }` (bypasses the expiry-buffer check, goes straight to lock + refresh).
2. Retry the original API call **once** with the fresh token.
3. If the retry succeeds, return the result.
4. If it still fails:
   - Token manager threw with `needsReauth: true` → return `{ needs_reauth: true }` (genuine revocation; now it's safe to tell the user to reconnect).
   - Otherwise → return HTTP 502 with `{ needs_reauth: false }` (transient; tell the user "retrying shortly").

**Never map a raw 401/403 straight to a "re-authenticate" prompt.** Trust the `needs_reauth` signal after the force-refresh+retry has run.

## Full reference

The complete flow (step-by-step token-manager logic, retry helper, proactive cron refresh via `pg_cron` + non-blocking `EdgeRuntime.waitUntil()`, rate limiting, OAuth-callback state reset, webhook error classification, and the pitfalls table) is in:

→ [oauth-refresh-reference.md](./oauth-refresh-reference.md)

## Quick checklist

- [ ] Tokens stored in Supabase, read/written only from Edge Functions; nothing token-related in the browser or committed `.env`.
- [ ] Every query scoped by `practice_id` + `archive_id`; no cross-practice token reuse.
- [ ] Refresh persisted with `??`, never `||`.
- [ ] Row-level lock on `refresh_locked_at` (2-min stale threshold); lock released in `finally`.
- [ ] Lock-wait path re-validates `expires_at` before returning.
- [ ] 401/400/403 → `status: "revoked"` + `needsReauth`; never retried.
- [ ] 429/5xx → retry with exponential backoff.
- [ ] API 401 → force-refresh + single retry before any re-auth prompt.
- [ ] OAuth callback resets `status: "active"` and `refresh_locked_at: null`.
