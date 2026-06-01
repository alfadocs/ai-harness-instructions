# Engineering Standard — AlfaDocs apps

Rules for every AlfaDocs app; task playbooks are in the **Skills** (indexed below).

Default to **more files, smaller files, narrower types**. Monoliths require justification.

## 1. Modularity & Layering

**Size budgets**
- Components ≤150 lines, ≤5 local state slots. Hooks/modules ≤80 lines, one concern.
- Functions ≤30 lines, ≤3 positional params (use a typed options object beyond that).
- One exported unit per file. Helpers used by only one file go in a colocated `./helpers.ts`.
- Touching an over-budget file? Split it first, then make the change.

**Layer direction (one-way):** `pages → feature components → hooks → services → repositories → integrations`. Pages orchestrate layout only. Components consume hooks; never call transport. Hooks call services; services call external APIs and return typed domain models. Lower layers never import from higher ones.

**Composition:** repeated UI → `.map` over a typed config array; forms → one schema; many small hooks beat one branchy mega-hook. Module-level singletons for services/repos — never `new Service()` inside a component. `useEffect`/`useMemo`/`useCallback` must list complete deps.

## 2. Reuse Before Creating

Always check for existing functions, hooks, services, or edge functions before writing new ones. Extend or compose what's there. Code duplication is rejected on review.

## 3. Type Safety & Boundary Validation

**TypeScript baseline** — keep `tsconfig.json` strict; never weaken `strict`/`no*` flags. Full config → `alfadocs-typescript-setup` skill.

No `any`/`as`/`@ts-ignore`. Prefer `unknown` and narrow. `array[i]` and `record[key]` are possibly `undefined`. Every service method has an explicit return type. Validate every external response with Zod at the service boundary — no `x.id ?? x.Id` fallback chains. All branches return; switches are exhaustive. Use `@/*` alias — no `../../../`. Remove unused imports (no `_` prefix).

## 4. Multi-Tenant Isolation

Every read/write/update/delete scoped by tenant. No cross-tenant leakage. Handlers verify identity and authorize the tenant ID **before any work** — never trust IDs from the request body. RLS on every tenant table.

## 5. XSS / CSP

Implement the strictest possible Content Security Policy.
- No inline scripts/styles unless nonced. No `unsafe-eval`, no `unsafe-inline`.
- Whitelist `img-src`, `connect-src`, `script-src`, `style-src`, `font-src` to known origins only.
- Replace generic OG/Twitter images with project-branded assets.
- Sanitize anything rendered as HTML; prefer text rendering by default.

## 6. External API Calls

- Centralize headers, base URLs, retry policy, and auth in one client per integration. Never inline `fetch` with hand-rolled headers in components or hooks.
- Refresh and use auth tokens within a single request scope to avoid race conditions across concurrent contexts.
- Set explicit timeouts on every outbound call.

## 7. Webhooks (general)

- Respond with the exact HTTP status code the sender expects. A wrong status is treated as failure by many providers.
- Do not assume JSON. Detect and parse the documented content type (often `application/x-www-form-urlencoded`, indexed brackets, signed multipart, etc.).
- Multiple events may arrive per call. Handle batches.
- Verify signatures before processing. Reject unsigned or invalid payloads with 4xx.
- Look up tenants by the identifier the provider actually sends.
- Handlers are idempotent — same payload, same effect.

## 8. Backend / Edge Function Hygiene

- Authenticate then authorize in the first lines of the handler — before logging or data access.
- Validate inputs with a schema; reject malformed input with 4xx.
- Return generic error messages. **Never** return stack traces or upstream error bodies.
- Use shared helpers for CORS, auth, error formatting — do not inline.
- Restrict CORS to known origins. Never `*`. **Reject non-matching origins with 403** — don't just omit the header.
- Encrypt sensitive data at rest. Never log secrets, tokens, or PII.
- Rate-limit public endpoints.

## 9. Error Handling

- Hooks expose `{ data, isLoading, error }`. Never swallow.
- Services throw **typed errors** (`ApiError`, `AuthError`, `ValidationError`), not stringified ones.
- UI renders empty / loading / error / success as first-class states.
- No empty `catch {}`. No `console.*` in committed code — use the project logger.

## 10. Constants & Naming

- No magic strings or numbers. Centralize storage keys, endpoint paths, query keys, limits, timeouts, retry counts.
- Names describe intent. Booleans are predicates (`isActive`, `hasAccess`, `canEdit`). Hooks start with `use`.
- Default to **no comments**. Add one only when the *why* is non-obvious. Never narrate the *what*.

## 11. When Integrating with AlfaDocs

Specializes §4, §6, §7 for the AlfaDocs case.

**Tenant scope** — `practiceId` and `archiveId` together define the tenant. Apply §4 with both columns: every query scoped, RLS on both, edge functions authorize the caller's `practiceId`/`archiveId` against their JWT before any work.

**API requests** — auth is **one scheme per request, never both**: **OAuth** sends `Authorization: Bearer <token>`; **API-key** sends `X-Api-Key: <key>`; both also send `Accept: application/json`. Base URL `https://app.alfadocs.com/api/v1`; call `/me` first for `practiceId` + `archiveId`. Secrets (API key / OAuth Client Secret) live in **Supabase Edge Function secrets** — never in the frontend or git.

**Token expiry** — OAuth tokens expire (~1h). Store `expires_at` as an absolute timestamp; check before every API call and refresh if expired (→ `oauth-token-refresh` skill). Never skip this in polling/long-running apps.

**Multi-practice token safety** — never use Practice A's token to call Practice B's endpoint. Scope token refresh + use to a single practice context per request and re-read the token immediately before the upstream call to defeat refresh races.

**Webhooks from AlfaDocs**
- Always respond **HTTP 200**. Never 202 — AlfaDocs treats it as an error.
- Content-Type is `application/x-www-form-urlencoded`, **not JSON**.
- Keys use indexed bracket notation (e.g. `0[event]`, `0[data][created_entity][archiveId]`); one call may carry multiple events. Parse with `groupFormDataToEvents`.
- Practice lookup uses `archiveId` from the payload, not `practiceId` directly.

## 12. UI — AlfaDocs Design System

Build **all** UI with **`@alfadocs/ui-kit`** (public npm; React 18 & 19). Never use Material UI, Bootstrap, shadcn, Chakra, Mantine, or hand-rolled CSS.

- Import tokens **once** at the root (`import '@alfadocs/ui-kit/tokens'`); wrap the tree in `ThemeRoot` + `I18nextProvider` + `TooltipProvider` + `Toaster`.
- Import components **per path** (`import { Button } from '@alfadocs/ui-kit/button'`), not the barrel; providers come from the barrel.
- Use the **dedicated** primitive — `PhoneInput`, `OtpInput`, `DataTable`, `AlertDialog` (destructive), `EmptyState`, `ContactCard`, `SignInWithAlfadocsButton` — never a generic `TextInput`/`Button`/`Dialog` when a specialised one exists.
- **Never invent `var(--…)` names** — `--color-background`, `--color-danger`, `--color-text-muted` etc. **do not exist**; they silently fall to hex and break all themes. Real tokens: `--background`, `--foreground`, `--muted-foreground`, `--card`, `--primary`, `--border`, `--ring`, `--destructive`, `--success`, `--warning`, `--error`.
- **No `style={{}}`** — Tailwind + `var(--…)` tokens only. No inline styles, no raw hex.
- **Delete `src/components/ui/`** on every new Lovable project (shadcn boilerplate — unused, import from `@alfadocs/ui-kit` only).
- Logical properties (RTL); strings via `useTranslation()`; raw HTML via `SafeHtml`. Four themes via `ThemeRoot`.
- **Marketplace app?** `MarketplaceAppShell` + `ConnectWithAlfadocs` (kit 0.36+) — don't hand-roll the frame/connect screen.
- → skill: `alfadocs-design-system`

## 13. Anti-Patterns — Reject on Sight

- `any`, `Promise<any>`, `as any`, `@ts-ignore`
- `new Service()` inside a component or hook body
- Component > 150 lines or > 5 local state slots
- `useEffect(..., [])` calling closures whose identity isn't stable
- Field-name fallback chains
- Two JSX blocks differing only in props
- Handlers reading tenant IDs (`practiceId`/`archiveId`) from input without authorizing them
- Returning 202 (or any non-200) to AlfaDocs webhooks
- Hand-rolled `fetch` with inline auth in UI code
- `console.log` in committed code
- A UI library other than `@alfadocs/ui-kit`; importing from `@/components/ui/` (shadcn boilerplate)
- Inline `style={{}}` or invented `var(--color-*)` token names — use real kit tokens + Tailwind
- Raw hex/rgb in styles; a generic primitive where a dedicated kit one exists

**Default: smaller, narrower, more composable.**

## Skills (load on demand)

All names are prefixed `alfadocs-`.

**Build** — `connected-app-oauth` (multi-practice OAuth BFF) · `connected-app-api-key` (single-practice) · `design-system` (§12) · `webhooks` (§7, §11) · `oauth-token-refresh` (§6, §11) · `gdpr-consent` · `patient-deep-link` · `typescript-setup` (§3 tsconfig).

**Review — before shipping** — `sanity-check` (fast "does it run & connect?" — run first) · `code-review` (§1, §2, §9, §10, §13) · `security-review` (§5, §8, §11) · `design-review` (§12) · `logic-review` (§4, §7, §11) · `architecture-review` (§1) · `accessibility-review`.
