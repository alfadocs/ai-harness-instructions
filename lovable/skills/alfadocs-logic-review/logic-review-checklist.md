# AlfaDocs logic review — failure modes & how to spot them

Companion to `SKILL.md`. Each row is a concrete defect, where it hides in the code, and the severity to assign. Use it as a grep/skim guide while auditing the Edge Functions, token manager, and webhook handler.

## practiceId / archiveId scoping

| What to check | How to spot the defect in code | Severity if wrong |
|---|---|---|
| `/me` called first | No `GET .../api/v1/me`; `practiceId`/`archiveId` hard-coded or read from a request param | blocker |
| Both IDs used | A data call path missing `{practiceId}` or `{archiveId}` segment | blocker |
| IDs resolved server-side | Edge Function reads `practiceId`/`archiveId` from `req` body/query instead of `/me` | blocker |
| Headers complete | Missing `Accept: application/json`, or the method's one auth header (`Authorization: Bearer` for OAuth, `X-Api-Key` for API-key — not both) | warning |
| Calls server-side only | `fetch('https://app.alfadocs.com/...')` in `src/` frontend code | blocker |

## Tenant isolation

| What to check | How to spot it | Severity |
|---|---|---|
| Rows carry `practice_id` | A mirrored table with no `practice_id`/`archive_id` column | blocker |
| Queries filter by practice | `.from('x').select()` with no `.eq('practice_id', …)` on a tenant table | blocker |
| Joins stay in-tenant | A join that doesn't constrain both sides to the same `practice_id` | blocker |
| RLS locks the table | Policy allowing `anon`/`authenticated` SELECT/INSERT on practice data | blocker |
| No token/ID reuse | A cached `practiceId`/token used across requests for different practices | blocker |

## Webhooks

| What to check | How to spot it | Severity |
|---|---|---|
| Raw body, not JSON | `await req.json()` in the webhook function | blocker |
| Bracket keys expanded | No `URLSearchParams` parse / no `\[([^\]]+)\]` walk | blocker |
| All events processed | Code reads index `0` only, ignores `1,2,…` | blocker |
| Route on `archiveId` | Looks for `practiceId` in the payload, or routes on `objectId` | blocker |
| Resolve to stored practice | No lookup table mapping `archive_id` → `practice_id` | warning |
| Returns 200 | `status: 202`, or returns 500 on a handler throw | blocker |
| Per-event try/catch | One bad event aborts the whole call / yields non-200 | blocker |
| Unknown archive tolerated | Missing `archiveId` throws instead of skip-and-200 | warning |
| Service-role DB access | Uses anon client inside the function | warning |

## Token refresh (OAuth)

| What to check | How to spot it | Severity |
|---|---|---|
| Row lock before refresh | `/oauth2/token` called with no `refresh_locked_at` guard | blocker |
| Atomic lock acquire | Lock set with a separate read-then-write (race) instead of a conditional update | blocker |
| Lock scoped to practice | Lock update missing `.eq('archive_id', …)`/`.eq('practice_id', …)` | blocker |
| `??` not `||` | `data.refresh_token || row.refresh_token` | blocker |
| Lock released | No `finally { … refresh_locked_at: null }` | warning |
| Stale-lock recovery | No `lt.<staleCutoff>` clause; a crashed refresh wedges the row forever | warning |
| Wait-path re-validates | Returns a token without re-checking `expires_at` after waiting | blocker |
| Error classes | 401/400/403 retried, or not marked `revoked` | blocker |
| 401 → force-refresh+retry | Raw 401 mapped straight to a reconnect prompt | warning |
| Tokens server-side only | `refresh_token`/`access_token` sent to browser, in `VITE_*`, or in git `.env` | blocker |

## Auth pattern choice

| Situation | Correct pattern | Defect |
|---|---|---|
| Many practices / users log in | OAuth 2.0 + PKCE BFF (Edge Function) | shipping one shared API key → blocker |
| Single practice, no login | One archive API key in `ALFADOCS_API_KEY` secret | building full OAuth it doesn't need → warning |
| Either | Secret only in Edge Function secrets | secret in frontend/.env/git → blocker |

## Response handling

| What to check | How to spot it | Severity |
|---|---|---|
| Pagination followed | List/count logic assumes one page; no loop over pages / `next` cursor | blocker if counts/lists matter |
| Empty handled | Map/iterate over results with no empty guard → crash or blank-broken UI | warning |
| Errors checked | `await res.json()` with no `res.ok` / status check first | blocker |
| Endpoints verified | Paths that don't match `app.alfadocs.com/api.html` | warning |

## Idempotency

| What to check | How to spot it | Severity |
|---|---|---|
| Replays don't duplicate | `INSERT` on every event with no unique key / upsert | blocker |
| Stable dedupe key | Upsert keyed on a local auto-id instead of AlfaDocs entity `id` + `practice_id` | blocker |
| Side effects guarded | Email/notification/downstream write fires unconditionally per event | blocker |

## Severity legend

- **blocker** — correctness or isolation defect; fix before shipping (and fix in place during the review).
- **warning** — should fix; degraded resilience or wrong-but-not-fatal design.
- **note** — hardening suggestion; safe to defer.
