The public AlfaDocs API has a rate limit of 10 requests per second.

# Engineering Standard — AlfaDocs apps

Rules for every AlfaDocs app; task playbooks are in the **Skills** (indexed below). Default to **more files, smaller files, narrower types**. Monoliths require justification.

## 1. Modularity & Layering

**Size budgets** — components ≤150 lines & ≤5 state slots; hooks/modules ≤80 lines, one concern; functions ≤30 lines & ≤3 positional params (typed options object beyond that); one exported unit per file (single-use helpers in a colocated `./helpers.ts`). Over-budget file? Split it first, then change it.

**Layer direction (one-way):** `pages → feature components → hooks → services → repositories → integrations`. Pages orchestrate layout; components consume hooks, never transport; services call external APIs and return typed domain models. Lower layers never import from higher.

**Composition:** repeated UI → `.map` over a typed config array; forms → one schema; many small hooks beat one branchy mega-hook. Module-level singletons for services — never `new Service()` in a component. `useEffect`/`useMemo`/`useCallback` list complete deps.

## 2. Reuse Before Creating

Check for existing functions, hooks, services, or edge functions before writing new ones. Extend or compose what's there — duplication is rejected on review.

## 3. Type Safety & Boundary Validation

Keep `tsconfig.json` strict; never weaken `strict`/`no*` flags (full config → `alfadocs-typescript-setup`). No `any`/`as`/`@ts-ignore` — prefer `unknown` and narrow. `array[i]`/`record[key]` are possibly `undefined`. Every service method has an explicit return type. Validate every external response with Zod at the service boundary — no `x.id ?? x.Id` fallback chains. All branches return; switches exhaustive. Use `@/*` alias, not `../../../`. No unused imports.

## 4. Multi-Tenant Isolation

Every read/write/update/delete scoped by tenant; no cross-tenant leakage. Handlers verify identity and authorize the tenant ID **before any work** — never trust IDs from the request body. RLS on every tenant table.

## 5. XSS / CSP

Strictest possible CSP: no inline scripts/styles unless nonced; no `unsafe-eval`/`unsafe-inline`; whitelist `img-src`/`connect-src`/`script-src`/`style-src`/`font-src` to known origins. Replace generic OG/Twitter images with project-branded assets. Sanitize anything rendered as HTML; prefer text by default.

## 6. External API Calls

- **Confirm the contract before mapping.** Check the API docs (AlfaDocs: <https://app.alfadocs.com/api.html>) for the exact path, params, response **envelope** (`{data}` vs `{results}` vs bare array) and **field names**, then map a real response. Guessing — `a.startsAt ?? a.starts_at`, or assuming an embedded object where the API returns an id — is a defect, not defensiveness.
- Treat a library's documented return type as the contract — a value may be a **string** where you assumed an object (e.g. an auth callback's `accessToken`); a wrong guess fails *silently* and stores `undefined`.
- Centralize headers, base URLs, retry, and auth in one client per integration — never inline `fetch` with hand-rolled headers in components/hooks.
- Refresh + use tokens within one request scope. With refresh-token **rotation**, serialize refreshes behind a real lock (single-flight/conditional update) — parallel refreshes trade rotated-away tokens and revoke the session. A lock column nothing reads is not a lock.
- Explicit timeouts on every outbound call.

## 7. Webhooks (general)

- Respond with the exact HTTP status the sender expects — a wrong status reads as failure.
- Don't assume JSON; parse the documented content type (often `application/x-www-form-urlencoded`, indexed brackets, signed multipart).
- Multiple events may arrive per call — handle batches.
- Verify signatures before processing; reject unsigned/invalid payloads with 4xx.
- Look up tenants by the identifier the provider actually sends. Handlers idempotent — same payload, same effect.

## 8. Backend / Edge Function Hygiene

- Authenticate then authorize in the first lines — before logging or data access.
- Validate inputs with a schema; reject malformed input with 4xx.
- Return generic errors — **never** stack traces or upstream error bodies.
- Shared helpers for CORS/auth/error formatting — don't inline.
- Restrict CORS to known origins, never `*`; **reject non-matching origins with 403** (don't just omit the header).
- Encrypt sensitive data at rest. Never log secrets, tokens, or PII. Rate-limit public endpoints.

## 9. Error Handling

- Hooks expose `{ data, isLoading, error }`; never swallow.
- Services throw **typed errors** (`ApiError`, `AuthError`, `ValidationError`), not stringified ones.
- UI renders empty/loading/error/success as first-class states.
- No empty `catch {}`; no `console.*` in committed code — use the project logger.

## 10. Constants & Naming

- No magic strings/numbers — centralize storage keys, endpoint paths, query keys, limits, timeouts, retries.
- Names describe intent; booleans are predicates (`isActive`, `hasAccess`); hooks start with `use`.
- Default to **no comments**; add one only when the *why* is non-obvious, never to narrate the *what*.

## 11. When Integrating with AlfaDocs

Specializes §4, §6, §7 for AlfaDocs.

**Tenant scope** — `practiceId` + `archiveId` together define the tenant. Apply §4 on both: every query scoped, RLS on both, edge functions authorize the caller's `practiceId`/`archiveId` against their JWT before any work.

**API requests** — one auth scheme per request, never both: **OAuth** → `Authorization: Bearer <token>`; **API-key** → `X-Api-Key: <key>`; both also `Accept: application/json`. Base URL `https://app.alfadocs.com/api/v1`. **Confirm every endpoint's fields, params, and envelope at <https://app.alfadocs.com/api.html> before mapping — don't guess.** Call `/me` first for `practiceId` + `archiveId` (nested under `.data`). Secrets (API key / Client Secret) live in **Supabase Edge Function secrets** — never the frontend or git.

**Token expiry** — OAuth tokens expire (~1h). Store `expires_at` absolute; check before every call and refresh if expired (→ `oauth-token-refresh`). Never skip in polling/long-running apps.

**Multi-practice token safety** — never use Practice A's token for Practice B. Scope refresh + use to one practice per request; re-read the token immediately before the upstream call to defeat refresh races.

**Webhooks from AlfaDocs**
- Always respond **HTTP 200** — never 202 (AlfaDocs treats it as an error).
- Content-Type is `application/x-www-form-urlencoded`, **not JSON**.
- Keys use indexed brackets (`0[event]`, `0[data][created_entity][archiveId]`); one call may carry multiple events — parse with `groupFormDataToEvents`.
- Practice lookup uses `archiveId` from the payload, not `practiceId`.

## 12. UI — AlfaDocs Design System

Build **all** UI with **`@alfadocs/ui-kit`** (public npm; React 18 & 19). Never Material UI, Bootstrap, shadcn, Chakra, Mantine, or hand-rolled CSS.

- Import tokens **once** at root (`import '@alfadocs/ui-kit/tokens'`); wrap the tree in `ThemeRoot` + `I18nextProvider` + `TooltipProvider` + `Toaster`.
- Import components **per path** (`import { Button } from '@alfadocs/ui-kit/button'`); providers from the barrel.
- Use the **dedicated** primitive — `PhoneInput`, `OtpInput`, `DataTable`, `AlertDialog` (destructive), `EmptyState`, `ContactCard`, `SignInWithAlfadocsButton` — never a generic `TextInput`/`Button`/`Dialog` when one exists.
- **Never invent `var(--…)` names** (`--color-background`, `--color-danger`… don't exist — they silently fall to hex and break themes). Real tokens: `--background`, `--foreground`, `--muted-foreground`, `--card`, `--primary`, `--border`, `--ring`, `--destructive`, `--success`, `--warning`, `--error`.
- **No `style={{}}`** — Tailwind + `var(--…)` tokens only; no raw hex.
- **Delete `src/components/ui/`** on every new Lovable project (shadcn boilerplate — import from `@alfadocs/ui-kit` only).
- Logical properties (RTL); strings via `useTranslation()`; raw HTML via `SafeHtml`; four themes via `ThemeRoot`.
- **Marketplace app?** `MarketplaceAppShell` + `ConnectWithAlfadocs` (kit 0.36+) — don't hand-roll the frame/connect screen.
- → skill: `alfadocs-design-system`

## 13. Anti-Patterns — Reject on Sight

- `any`, `Promise<any>`, `as any`, `@ts-ignore`
- `new Service()` inside a component/hook body
- Component > 150 lines or > 5 state slots
- `useEffect(..., [])` calling closures whose identity isn't stable
- Field-name fallback chains / mapping an endpoint whose shape you haven't confirmed at `app.alfadocs.com/api.html` (guessing fields or envelope)
- Two JSX blocks differing only in props
- Handlers reading tenant IDs (`practiceId`/`archiveId`) from input without authorizing them
- Returning 202 (or any non-200) to AlfaDocs webhooks
- Hand-rolled `fetch` with inline auth in UI code; `console.log` in committed code
- A UI library other than `@alfadocs/ui-kit`; importing from `@/components/ui/`
- Inline `style={{}}` or invented `var(--color-*)` names; raw hex/rgb; a generic primitive where a dedicated kit one exists

**Default: smaller, narrower, more composable.**

## Skills (load on demand)

All names prefixed `alfadocs-`.

**Build** — `connected-app-oauth` (multi-practice OAuth BFF) · `connected-app-api-key` (single-practice) · `staff-sso-app` · `google-workspace-auth` · `prototype-app` · `design-system` (§12) · `webhooks` (§7, §11) · `oauth-token-refresh` (§6, §11) · `gdpr-consent` · `patient-deep-link` · `typescript-setup` (§3).

**Review — before shipping** — `sanity-check` (run first) · `code-review` (§1, §2, §9, §10, §13) · `security-review` (§5, §8, §11) · `design-review` (§12) · `logic-review` (§4, §7, §11) · `architecture-review` (§1) · `accessibility-review`.
