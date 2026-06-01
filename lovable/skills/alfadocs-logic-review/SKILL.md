---
name: alfadocs-logic-review
description: Use when reviewing or auditing the business-logic correctness and data flow of an AlfaDocs integration — verifying practiceId/archiveId scoping, tenant isolation, webhook parsing and responses, concurrency-safe token refresh, the right OAuth-vs-API-key choice, and idempotent/resilient handling of API responses and repeated events. Load this for a correctness review, NOT for building from scratch.
---

# AlfaDocs integration — business-logic correctness review

> **Enforces Engineering Standard** (workspace knowledge) §4, §7, §11. Cite the section number when you flag a violation.

A correctness audit of an AlfaDocs app (Lovable + Supabase). The UI can be perfect and the app still silently leak one practice's data into another, drop webhooks, or brick an OAuth connection. This playbook checks the **logic and data flow**, not the styling.

Work through each section against the actual code (Edge Functions, token manager, webhook handler, any DB queries). For each finding, mark **blocker / warning / note**, cite the file + area, **fix blockers**, and flag the rest. Reporting format at the bottom.

The companion checklist with the exact failure modes and how to spot them in code is in **[logic-review-checklist.md](./logic-review-checklist.md)** — open it while reviewing.

---

## 1. `/me` → practiceId + archiveId on every call

- [ ] App calls `GET /me` (on `https://app.alfadocs.com/api/v1`) **first** and reads `practiceId` **and** `archiveId` from it. Both, not just one.
- [ ] **Every** subsequent AlfaDocs call uses the scoped path `/practices/{practiceId}/archives/{archiveId}/…`. No call hits an unscoped or hard-coded path. (**blocker** if any data call omits either ID.)
- [ ] `practiceId`/`archiveId` are derived **server-side** from `/me` (using the token/key in the Edge Function) — never accepted from a browser request parameter. A frontend must not be able to choose which practice it reads. (**blocker** if the function trusts a client-supplied `practiceId`/`archiveId`.)
- [ ] The correct auth header for the app's method on every call — `Authorization: Bearer <token>` for **OAuth**, **or** `X-Api-Key` for an **API key** (mutually exclusive — never both) — plus `Accept: application/json`.
- [ ] All AlfaDocs calls originate in an Edge Function, never the browser.

## 2. Tenant isolation — everything scoped by practiceId

- [ ] Every stored row that mirrors AlfaDocs data carries the `practice_id` (and `archive_id`) it belongs to.
- [ ] Every read/write/join filters by `practice_id` — no query can return rows from another practice. Watch for `.select()` with no practice filter, or a filter on user/session but not practice. (**blocker** for any cross-practice read or write path.)
- [ ] RLS denies `anon` and `authenticated`; only the service role (inside Edge Functions) touches these tables. (**blocker** if a practice-data table is readable by `anon`/`authenticated`.)
- [ ] No code path reuses one practice's token, key, or resolved IDs to serve another practice's request.

## 3. Webhooks — parse, route, respond

- [ ] Handler reads `await req.text()`, **not** `await req.json()` (payload is `application/x-www-form-urlencoded`). (**blocker** — `req.json()` throws on every delivery.)
- [ ] Indexed bracket keys are expanded into structured events, grouped by the leading index `0,1,2,…` — **multiple events per POST** are all processed, not just index 0. (**blocker** if only the first event is handled.)
- [ ] Routing key is `archiveId` pulled from inside `data.{created,updated,deleted}_entity`, resolved to the stored `practiceId`. The webhook does **not** supply `practiceId`. (**blocker** if it expects `practiceId` in the payload or routes on the wrong key.)
- [ ] Handler returns **HTTP 200** — never **202** (AlfaDocs treats 202/any non-200 as failure and retries/alerts). (**blocker** — this is the single most common defect.)
- [ ] A processing error on one event is caught and logged; the call still returns 200. Unknown/missing `archiveId` → log and skip that event, still 200 overall.
- [ ] Webhook DB access uses the service role; webhook tables stay locked from `anon`/`authenticated`.

## 4. Token refresh — concurrency-safe & isolated (OAuth apps)

- [ ] Refresh is guarded by a **row-level lock** (`refresh_locked_at`, ~2-min stale threshold), acquired atomically before calling `/oauth2/token`, released in `finally`. Two concurrent requests must not both refresh. (**blocker** if refresh runs without a lock — rotation double-refresh permanently bricks the connection.)
- [ ] The lock is scoped to **this** `practice_id` + `archive_id`. Refresh never operates on, or hands back, another practice's row. (**blocker** for any cross-practice token use.)
- [ ] Rotated refresh token persisted with `??`, **never** `||` (an empty-string field with `||` falls back to the revoked old token). (**blocker** — `||` here is silently fatal.)
- [ ] Lock-wait path re-validates `expires_at` is in the future before returning a token; never returns a known-stale token.
- [ ] Error classification correct: 401/400/403 → mark row `revoked` + `needsReauth`, do **not** retry; 429/5xx → retry with backoff.
- [ ] An API 401 triggers force-refresh + a **single** retry before any re-auth prompt — a raw 401 is never mapped straight to "reconnect".
- [ ] Tokens live only in Supabase, read/written only from Edge Functions. No token in the browser, in a `VITE_*` var, or committed `.env`. (**blocker** if a refresh/access token reaches the frontend.)

## 5. Right auth pattern for the job

- [ ] **Many practices / end users log in with their own AlfaDocs account** → OAuth 2.0 + PKCE BFF (Edge Function). (**blocker** if a multi-practice app ships a single shared API key.)
- [ ] **One practice's own data, no user login** (internal dashboard/report/automation) → a single archive **API key** in the `ALFADOCS_API_KEY` Edge Function secret. (**warning/blocker** if a single-practice tool drags in a full OAuth flow it doesn't need — over-engineering that adds failure surface.)
- [ ] Either way the secret (client secret or API key) is **only** in Edge Function secrets, never frontend/.env/git.

## 6. Resilient response handling

- [ ] **Pagination** is followed where the endpoint paginates — the app doesn't silently process only page 1. (**blocker** if "total" logic or list display assumes one page.)
- [ ] **Empty** results are handled (no patients, no appointments) without crashing or showing a broken state.
- [ ] **Error / non-2xx** responses from AlfaDocs are checked before parsing `.json()` and surfaced sensibly — not assumed to be success. (**blocker** if `res.json()` is parsed without a status check and an error body is treated as data.)
- [ ] Endpoint paths/methods/body shapes were verified against `https://app.alfadocs.com/api.html`, not guessed.

## 7. Idempotency — repeated events & retries

- [ ] Webhook/event handling is **idempotent**: AlfaDocs retries on non-200, and may resend, so the same `event` + `objectId` arriving twice must not double-create / double-charge / double-notify. (**blocker** if a retry produces a duplicate row or duplicate side effect.)
- [ ] Upserts key on a stable AlfaDocs identifier (e.g. entity `id` + `practice_id`), not on an auto-generated local key, so a replay updates rather than duplicates.
- [ ] Any external side effect triggered by an event (email, message, downstream write) is guarded against re-firing on replay.

---

## How to report findings

Produce one merged list, grouped by severity. For each item give the **file + area** and a one-line fix.

- **Blocker** — correctness/isolation defects that must be fixed before shipping: missing practice scoping, cross-practice token/data reuse, webhook returning 202 / parsing as JSON / handling only the first event, refresh without a row lock or with `||`, a token reaching the frontend, non-idempotent event handling, wrong auth pattern for a multi-practice app. **Fix these in place.**
- **Warning** — should fix: unfollowed pagination, unhandled empty/error responses, an over-heavy auth choice, missing stale-lock handling, weak idempotency keys.
- **Note** — observations and hardening suggestions (extra logging, defensive checks, reconciliation jobs).

End with a one-line verdict: **zero blockers → safe to ship on logic grounds**, otherwise list the blockers (with file + line) that remain.
