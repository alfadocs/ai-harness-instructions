# Audit checks — scan patterns

Concrete things to grep/inspect for each section of `SKILL.md`. These find the
common failures fast; a clean grep is not proof, but a hit is almost always a
finding. Treat the source tree, the `.env*` files, the **full git history**, the
Supabase migrations/policies, and the Edge Functions as in-scope.

---

## 1. Secrets server-side only

Search the whole tree (and git history) for credential names and key shapes:

```bash
# Credential names that must NEVER be client-side / committed
git grep -nIE 'ALFADOCS_(CLIENT_SECRET|API_KEY|CLIENT_ID)|CLIENT_SECRET|SERVICE_ROLE' -- . ':!supabase/functions'

# VITE_ vars that aren't the allowlisted Supabase ones = finding
git grep -nIE 'VITE_[A-Z_]+' -- '*.env*' '*.ts' '*.tsx' \
  | grep -vE 'VITE_SUPABASE_(URL|PUBLISHABLE_KEY|PROJECT_ID|ANON_KEY)'

# Any .env key outside the eight public Supabase keys
git grep -nIE '^[A-Z0-9_]+=' -- '*.env*' \
  | grep -vE '^[^:]+:(SUPABASE_URL|SUPABASE_PUBLISHABLE_KEY|SUPABASE_PROJECT_ID|SUPABASE_ANON_KEY|VITE_SUPABASE_URL|VITE_SUPABASE_PUBLISHABLE_KEY|VITE_SUPABASE_PROJECT_ID|VITE_SUPABASE_ANON_KEY)='

# Leaked key shapes anywhere in history
git log -p --all -S 'eyJ' -- . | grep -nE 'eyJ[A-Za-z0-9_-]{20,}'   # JWTs / Supabase keys
git grep -nIE '(sk|pk|rk)_(live|test)_[A-Za-z0-9]{16,}'              # generic secret-key shapes
```

**Allowlisted `.env*` keys (everything else is a finding):**
`SUPABASE_URL`, `SUPABASE_PUBLISHABLE_KEY`, `SUPABASE_PROJECT_ID`, `SUPABASE_ANON_KEY`,
`VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY`, `VITE_SUPABASE_PROJECT_ID`, `VITE_SUPABASE_ANON_KEY`.

If a real secret is found in history: **rotate it upstream** (regenerate the AlfaDocs
Client Secret / API key / Supabase service-role key), then remove from code. Assume
the old value is already public — deleting the file is not a fix.

---

## 2. Browser never holds a token

```bash
# AlfaDocs API called directly from the client = finding
git grep -nE 'app\.alfadocs\.com/api' -- 'src/**' '*.tsx' '*.ts' ':!supabase/functions'

# Tokens parked in browser storage
git grep -nIE "(localStorage|sessionStorage)\.(set|get)Item\(.*(token|access|refresh)" -- 'src/**'

# Wrong auth system for AlfaDocs login
git grep -nE 'supabase\.auth\.(signIn|signInWith)' -- 'src/**'

# React /callback route (the callback must be server-side)
git grep -nE 'path=["\x27]/callback' -- 'src/**'
```

Confirm Edge Functions read tokens via the session cookie and return only data,
never the raw token. Confirm refresh holds a row-level lock (see
`alfadocs-oauth-token-refresh`).

---

## 3. Supabase RLS

Inspect migrations / SQL for each tenant table:

```bash
git grep -nE 'create table|CREATE TABLE' -- 'supabase/**' '*.sql'
git grep -nE 'enable row level security|ENABLE ROW LEVEL SECURITY' -- 'supabase/**' '*.sql'
git grep -nE 'create policy|CREATE POLICY' -- 'supabase/**' '*.sql'
```

For every table holding AlfaDocs/tenant data confirm:
- RLS is **enabled**.
- A policy denies `anon` and `authenticated` (only the service role, used inside Edge
  Functions, reaches the rows). A policy with `using (true)` or `to public` /
  `to authenticated` granting access is a finding.
- The table has `practice_id` (and usually `archive_id`), and Edge-Function queries
  filter on it. A query with no practice scope is a finding.
- The frontend cannot supply `practiceId`/`archiveId` — they come from `/me` or the
  stored session inside the function.

---

## 4. XSS & CSP

```bash
# XSS sinks
git grep -nE 'dangerouslySetInnerHTML' -- 'src/**' | grep -v 'SafeHtml'
git grep -nE '\beval\(|new Function\(|\.innerHTML\s*=' -- 'src/**' 'supabase/**'

# CSP: find the meta/header and read it
git grep -nE 'Content-Security-Policy|http-equiv' -- 'index.html' 'src/**' 'supabase/**'
git grep -nE 'img-src[^;]*lovable\.dev' -- .         # must be removed
git grep -nE "script-src[^;]*('unsafe-inline'|'unsafe-eval'|\*)" -- .
```

Check OG/Twitter image meta tags point at AlfaDocs-branded assets, not Lovable
defaults.

---

## 5. OAuth callback & origin

```bash
# Origin must be the exact frontend, never "*"
git grep -nE 'Access-Control-Allow-Origin' -- 'supabase/**'
git grep -nE 'ALLOWED_ORIGIN' -- 'supabase/**'
git grep -nE "['\"]\*['\"]" -- 'supabase/functions/**'   # any "*" in functions is suspect

# state validation + X-Requested-With guard present in the auth/proxy functions
git grep -nE 'state' -- 'supabase/functions/alfadocs-auth/**'
git grep -niE 'x-requested-with' -- 'supabase/functions/**'
```

Manually confirm: registered Callback URL == `ALFADOCS_REDIRECT_URI` byte-for-byte
and ends in `/functions/v1/alfadocs-auth/callback`; one registration per environment.
Session cookie is `HttpOnly; Secure; SameSite`.

---

## 6. Dismissable-layer integrity

```bash
# Capture-phase global pointer listeners that stopPropagation break ui-kit overlays
git grep -nE "addEventListener\((['\"])(pointerdown|mousedown|click)\1.*true\)" -- 'src/**'
git grep -nE 'stopPropagation\(\)' -- 'src/**'
```

A capture-phase `document` `pointerdown`/`mousedown` handler that calls
`stopPropagation()` swallows the outside-press that Dialog/Popover/DropdownMenu/
Tooltip use to dismiss — users get stuck in the layer. Flag it; use the component's
own `onOpenChange` / dismiss handlers instead.

---

## 7. Webhooks

```bash
git grep -nE 'req\.json\(\)' -- 'supabase/functions/**webhook**'   # should be req.text() (form-urlencoded)
git grep -nE 'status:\s*202' -- 'supabase/functions/**'           # never 202
git grep -niE 'webhook.*(secret|signature|verify)' -- 'supabase/functions/**'
```

Confirm: a verification step (shared/signing secret from an Edge Function secret)
gates processing; routing is by `archiveId` from the payload, resolved to a stored
`practiceId`; a `practiceId` in the body is never trusted; handler always returns
`200`; unknown `archiveId` is logged and skipped.

---

## 8. Token / PII logging

```bash
git grep -nIE 'console\.(log|info|warn|error)\(' -- 'supabase/functions/**' 'src/**' \
  | grep -iE 'token|secret|api[_-]?key|password|access|refresh'
# PII fields in logs
git grep -nIE 'console\.(log|info|warn|error)\(' -- . \
  | grep -iE 'fiscalCode|taxId|dateOfBirth|dob|firstName|lastName|patientName|email|phone'
```

Confirm errors surfaced to the browser don't echo tokens, secrets, or raw upstream
error bodies.

---

## 9. Consent before processing patient data

See `alfadocs-gdpr-consent` for the full rule. Verify by inspection:

- A consent checkbox exists on every page that can send data out; not pre-ticked
  (`defaultChecked` / `checked={true}` initial state is a finding); send/export/sync
  buttons are `disabled` until ticked.
- A `consent_log` (or equivalent) table is **append-only**: revoke sets `revoked_at`,
  no `delete` on consent rows.
- The consent record is written from an Edge Function (server-side timestamp + user
  identity), and a server-side consent check runs before every outgoing data call —
  not just the browser checkbox.

```bash
git grep -nIE 'consent' -- 'src/**' 'supabase/**'
git grep -nIE 'defaultChecked|checked=\{true\}' -- 'src/**'   # pre-ticked consent = finding
git grep -nIE 'revoked_at|delete.*consent' -- 'supabase/**' 'src/**'
```
