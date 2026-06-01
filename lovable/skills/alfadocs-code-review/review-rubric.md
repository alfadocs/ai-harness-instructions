# AlfaDocs code-review — extended rubric

Rationale and worked examples behind the seven checklist dimensions in `SKILL.md`. Read this when a finding is borderline or you need the reasoning to justify a severity.

---

## Scope reminder

This is a **code-quality** review. Tenancy/RLS, secrets handling, OAuth BFF wiring, and design-system conformance each have their own dedicated skill and check — don't re-derive them here. When a quality finding *also* trips one of those rules (a logged token, a missing `practiceId`), say so, cite the rule, and treat it at the higher severity. Otherwise stay in your lane: is the code correct, lean, and maintainable?

Apps under review are Lovable + Supabase: a React/Vite frontend and TypeScript (Deno) Edge Functions. The AlfaDocs API is reached **only** from Edge Functions, never the browser.

---

## 1. Correctness — worked checks

- **Re-read the intent.** Open the prompt/feature description that drove the work and tick each requirement against actual code. A feature that compiles but quietly no-ops on the main path is the most expensive miss.
- **Async discipline.** A function that calls an `async` helper without `await` returns a pending promise — downstream code sees `undefined`/`[object Promise]`. Grep for fetches without `await` and `.then` chains without `.catch`.
- **Pagination.** AlfaDocs list endpoints page. A loop that reads page 1 and stops, or that never advances the cursor, silently truncates data. Confirm the loop terminates on a real end condition.
- **AlfaDocs correctness invariants** (these are *function-breaking*, not stylistic):
  - Every resource path is `/api/v1/practices/{practiceId}/archives/{archiveId}/<resource>` — both IDs present, fetched from `/me`.
  - Webhook responses are HTTP **200**. A `202` (or any non-200) makes AlfaDocs treat delivery as failed.
  - Webhook bodies are `application/x-www-form-urlencoded` parsed via `req.text()` — `req.json()` throws.
  - Batch loops space requests (~150 ms) to stay under 10 req/s.

## 2. Reuse / DRY — what to look for

The single most common AlfaDocs-app smell is **duplicated integration plumbing**:

- The AlfaDocs auth header(s) (`Authorization: Bearer` for OAuth **or** `X-Api-Key` for API-key, plus `Accept`) rebuilt inline in every function → extract `alfadocsHeaders(token)`.
- The webhook bracket-notation parser copied per handler → one shared `groupFormDataToEvents`.
- Token lookup/refresh logic re-implemented per proxy → one shared module; refresh stays server-side and uses the documented concurrency lock.
- A new Edge Function created when an existing one could take a parameter. **Before approving a new function, search for one that already does ~80% of it.** Reuse or extend with a stable signature; deprecate cleanly if you must change one.

When you flag duplication, always propose the concrete extraction (name the helper, name the file, name the call sites). "This is duplicated" without a fix is a weak finding.

React side: components that differ only by label/props → one parameterised component. Repeated literals (base URL, table names, scope strings) → shared constants.

## 3. Simplicity & readability

- Guard clauses beat nested `if/else` pyramids. `if (!session) return ...` at the top reads better than wrapping the whole body.
- One function, one job. Split fetch / transform / render.
- Delete speculative generality — config flags with one value, abstractions with one caller, options never passed.
- Comments explain *why*, not *what*. A comment restating the code is noise; a comment explaining a non-obvious AlfaDocs quirk ("200 not 202 — AlfaDocs treats 202 as failure") earns its place. Keep comments short; no ticket references.

## 4. Clear naming

| Smell | Better |
|---|---|
| `data`, `data2`, `res`, `tmp` | `patients`, `meResponse`, `connectedPractice` |
| `pid`, `aid` | `practiceId`, `archiveId` |
| `getPatients()` that also writes | `fetchAndCachePatients()` or split it |
| `flag`, `check`, `status` booleans | `isLoading`, `hasConsent`, `isWebhookVerified` |

Consistency matters as much as the individual name — match the convention already used in the file/repo.

## 5. Error handling — the swallowing patterns

Blockers:

```ts
try { await doThing(); } catch {}            // swallowed — no log, no recovery
const { data } = await supabase.from(...)...; // no error check, no empty check
return data.map(...)                          // explodes when data is null
```

Fix shape:

```ts
const { data, error } = await supabase.from("connected_practices").select(...);
if (error) { console.error("practice lookup failed", { archiveId, error }); return errorState(); }
if (!data?.length) return emptyState();        // handle the empty branch explicitly
```

- Every fetch handles non-2xx (`if (!res.ok) ...`), network throw, and empty body.
- User-facing failures surface a message (`Toaster`/error state), never a blank screen.
- **Never** put a token, secret, or full patient record in a log line — including inside a `catch`. Log identifiers and error messages, not payloads.

## 6. Dead code / leftover debug

- `console.log` of tokens, `/me` responses, patient objects → **blocker** (sensitive data leak + noise). Plain dev `console.log` → warning.
- Commented-out blocks, unused imports/vars, dead files, `debugger;`, orphaned TODO scaffolding → warning; remove them.
- Mock data / test routes / hardcoded fixtures must not ship to production.

## 7. Build

- `npm run build` clean — no errors, no **new** warnings. Run it; don't assume.
- Type escapes used to silence the compiler (`any`, `@ts-ignore`, `as unknown as`, `!`) are findings — they hide the real defect. Fix the type.
- No imports to deleted files, no env reads that crash at boot.

---

## Severity guide

- **Blocker** — wrong behaviour, swallowed error, logged token/secret/PHI, webhook 202, missing `practiceId`/`archiveId`, build failure. Fix in place, then report what changed.
- **Warning** — duplicated logic, redundant Edge Function, missing empty/error branch, weak naming, non-sensitive leftover `console.log`. Propose a concrete fix; apply if cheap and in scope.
- **Note** — readability and naming polish, minor simplifications. List and move on.

Close with a verdict line: **ship** / **hold**, counts per severity, and (if you fixed blockers) confirmation the build still passes.
