---
name: alfadocs-code-review
description: Use when reviewing, auditing, or cleaning up the code of an AlfaDocs/Lovable app before shipping — checking correctness, reuse/DRY, readability, error handling, dead code, leftover debug logging, and that the build still succeeds. Not for building features; load this to critique an app that's already (partly) built.
---

# AlfaDocs code-quality review

> **Enforces Engineering Standard** (workspace knowledge) §1, §2, §9, §10, §13. Cite the section number when you flag a violation.

A pre-ship review pass over an AlfaDocs app (Lovable frontend + Supabase Edge Functions). This skill covers **general code quality**. It does **not** re-derive the security / tenancy / design-system rules — those have their own checks (RLS + `practiceId` isolation, secrets in Edge Function secrets, OAuth BFF, `@alfadocs/ui-kit`). When a finding overlaps one of those, cite it and defer to the dedicated skill; here, focus on whether the code is **correct, lean, and maintainable**.

Walk the seven dimensions below. For each one: scan the relevant files, note concrete findings with a file/area reference, assign a severity, **fix the blockers**, and flag the rest. If you need the longer rationale or examples, see [review-rubric.md](./review-rubric.md).

---

## 1. Correctness

- Logic does what the prompt/feature intended — re-read the request and confirm each acceptance point is actually met, not just plausibly stubbed.
- Async paths are `await`ed; no floating promises, no `.then()` without a `.catch()`.
- Edge cases handled: empty arrays/lists, `null`/`undefined`, zero/negative numbers, missing optional fields, pagination not stopping early.
- Off-by-one, inverted conditionals, wrong comparison operators, `==` where `===` is meant.
- AlfaDocs specifics that are *correctness*, not just security: every API path carries both `practiceId` **and** `archiveId`; webhook handler returns **200 not 202**; the `~150 ms` spacing is present in batch loops (10 req/s limit). A missing `archiveId` or a 202 is a correctness blocker, not a style nit.

## 2. Reuse / DRY (be proactive)

- **Search before trusting that something is new.** Duplicated or near-duplicated blocks (a second copy of the AlfaDocs fetch headers, a re-implemented form-data parser, a hand-rolled token lookup) are findings — propose extracting a shared helper.
- **Edge Functions: reuse and extend, don't multiply.** Before accepting a new Edge Function, check whether an existing one already does the job (or could with a small parameter). A second `*-callback` or a third near-identical proxy function is a flag. Shared logic (auth header builder, `groupFormDataToEvents`, token refresh) belongs in **one** module imported by the rest.
- Copy-pasted React components that differ only in props/strings → propose one parameterised component.
- Repeated magic strings/numbers (API base URL, scopes, table names) → a single constant.
- When you flag duplication, **name the refactor**: "extract `alfadocsHeaders(token)` into `_shared/`, call it from both functions."

## 3. Simplicity & readability

- Prefer the boring solution. Flag needless abstraction, premature generalisation, and clever one-liners that hide intent.
- Deeply nested conditionals → early returns / guard clauses.
- Functions doing too much → split at the seams. A function that fetches, transforms, and renders is three functions.
- Dead config, unused props, unreachable branches.

## 4. Clear naming

- Names say what the thing is/does: `practiceId` not `pid`, `connectedPractices` not `data2`, `fetchPatients` not `getStuff`.
- No misleading names (a `get*` that mutates, a `list` that returns one item).
- Booleans read as predicates (`isLoading`, `hasConsent`), not `flag`/`status` blobs.
- Consistent casing/convention across the file and the repo.

## 5. Error handling (don't swallow)

- **Empty `catch {}` blocks are a blocker.** At minimum log with context; usually surface a user-facing error state too.
- Handle the **error and empty** branches of every API/DB call: non-2xx responses, network failure, `data: null`, zero-row results. A happy-path-only fetch is a finding.
- Don't `console.error` and then continue as if it succeeded — either recover or stop.
- User-facing failures show a real message (via `Toaster` / an error state), not a blank screen or a silent no-op.
- **Never log tokens, secrets, or full patient records** — even in a `catch`. This overlaps the security rule; treat a logged token as a blocker.

## 6. No dead code / no leftover debug

- Remove `console.log`/`console.debug` left from development — **especially any that print tokens, API responses, or patient data.** A token in a `console.log` is a blocker, not a note.
- Delete commented-out code blocks, unused imports, unused variables, unreferenced files, TODO stubs that were never wired up.
- No throwaway test routes, mock data, or hardcoded fixtures shipping to production.
- No `debugger;` statements.

## 7. Build succeeds

- The app builds clean: `npm run build` (and `tsc` if applicable) with **no errors and no new warnings**.
- No type holes papering over real issues — `any`, `as unknown as X`, `// @ts-ignore`, or `!` non-null assertions used to silence the compiler are findings; fix the underlying type.
- No unresolved imports, no references to deleted files, no missing env-var reads that crash at boot.

---

## How to report

Group findings by severity. For each: **state the issue, cite the file/area, and give the fix.**

- **Blocker** — must fix before shipping. Wrong output, swallowed errors, a logged token, a 202 webhook response, a missing `practiceId`/`archiveId`, or a broken build. **Fix these in place**, then note what you changed.
- **Warning** — should fix. Duplicated logic, a needless second Edge Function, missing empty/error handling, poor naming, leftover `console.log` (non-sensitive). Flag with a concrete suggested fix; fix if cheap and in-scope.
- **Note** — optional polish. Readability nudges, naming preferences, minor simplifications. List and move on.

End with a one-line verdict: **ship** (no blockers) or **hold** (blockers remain), and the count per severity. If you fixed blockers, re-confirm the build passes after your edits.
