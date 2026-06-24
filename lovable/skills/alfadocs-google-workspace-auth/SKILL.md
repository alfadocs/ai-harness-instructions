---
name: alfadocs-google-workspace-auth
description: Use when adding staff/employee login to an INTERNAL AlfaDocs app — sign in with a Google Workspace (@alfadocs.com) account via Supabase Auth, locked to the company domain, with server-side enforcement and RLS keyed to the email/role claim. NOT for "Sign in with AlfaDocs" customer login (that is OAuth — see alfadocs-connected-app-oauth), and NOT for single-practice API-key tools (alfadocs-connected-app-api-key).
---

# Staff login with Google Workspace (Supabase Auth)

This is the login for an **internal AlfaDocs tool** used by **AlfaDocs employees** — they sign in with their **`@alfadocs.com` Google Workspace** account. Identity and session are handled by **Supabase Auth** (Google provider), not by AlfaDocs OAuth.

Use this when the audience is AlfaDocs staff. Do **not** use it when end users are *customers* logging in with their own AlfaDocs accounts — that is "Sign in with AlfaDocs" (OAuth 2.0 + PKCE), a different skill (`alfadocs-connected-app-oauth`).

## The domain lock — three layers, strongest first

`@alfadocs.com`-only access is **not** something you enforce in the browser. Use real controls, in this order:

1. **Google Cloud OAuth consent screen = "Internal" (the real boundary).** Create the OAuth client in a Google Cloud project **owned by the alfadocs.com Google Workspace org** and set the consent screen **User Type → Internal**. Google then issues tokens **only** to `@alfadocs.com` accounts — external Google accounts can't even complete sign-in. This is the control that actually keeps strangers out.
2. **Server-side domain check (defense in depth).** A DB trigger (or a Supabase Auth Hook) rejects any non-`@alfadocs.com` user at creation, so even a misconfigured consent screen can't leak access.
3. **RLS predicates on the email/role claim (defense in depth).** Every table policy re-checks the domain, so data is unreadable without a valid `@alfadocs.com` session.

The `hd` sign-in hint below is **UX only** — it pre-selects the right account; it is *not* a security boundary. Never rely on it.

## 1. Configure the Supabase Google provider

In the Supabase dashboard → **Authentication → Providers → Google**: paste the **Client ID** and **Client Secret** from the Google Cloud project (the Workspace-owned, Internal one). The client secret lives **only** in Supabase provider config — never in the frontend or git.

Add your app's redirect URL to **Authentication → URL Configuration → Redirect URLs** (e.g. `https://<app>/auth/callback` and your local dev URL) or sign-in redirects are rejected.

## 2. Sign-in from the frontend

Use the kit's branded button — don't hand-roll a Google button (see `alfadocs-design-system`):

```tsx
import { SocialSignInButton } from '@alfadocs/ui-kit/social-sign-in-button';
import { supabase } from '@/lib/supabase';

async function signIn() {
  await supabase.auth.signInWithOAuth({
    provider: 'google',
    options: {
      redirectTo: `${window.location.origin}/auth/callback`,
      queryParams: { hd: 'alfadocs.com', prompt: 'select_account' }, // hd = UX hint only
    },
  });
}

// <SocialSignInButton provider="google" onClick={signIn} />
```

## 3. Enforce the domain server-side

A trigger on `auth.users` blocks any non-company account at creation:

```sql
create or replace function public.enforce_alfadocs_domain()
returns trigger language plpgsql security definer as $$
begin
  if new.email is null or lower(new.email) not like '%@alfadocs.com' then
    raise exception 'Only @alfadocs.com accounts are allowed';
  end if;
  return new;
end $$;

create trigger enforce_alfadocs_domain
  before insert on auth.users
  for each row execute function public.enforce_alfadocs_domain();
```

(Equivalent: a Supabase **"Before User Created" Auth Hook** that returns an error for non-matching domains. Pick one, not both.)

## 4. Lock down data with RLS

Deny by default, then allow only a valid `@alfadocs.com` session. Scope your **own** app tables — there is no `practiceId` tenant here (see `alfadocs-staff-sso-app`).

```sql
alter table app_notes enable row level security;

create policy "staff read"   on app_notes for select
  using ( lower(auth.jwt() ->> 'email') like '%@alfadocs.com' );

create policy "owner write"  on app_notes for all
  using ( user_id = auth.uid() )
  with check ( user_id = auth.uid() );
```

For role-based access, keep a `profiles` table (`id uuid references auth.users`, `role text`) and reference `role` in policies instead of (or alongside) the email check.

## 5. Guard the app in React

```tsx
const { data: { session } } = await supabase.auth.getSession();
if (!session) navigate('/login');

// keep it live:
supabase.auth.onAuthStateChange((_event, session) => {
  if (!session) navigate('/login');
});
```

Wrap the authenticated area in a `ProtectedRoute` that does this check; render the app chrome (`MarketplaceAppShell`) with the signed-in staff user.

## What stays server-side

- **Google client secret** → Supabase provider config only.
- **Supabase service-role key** → Edge Functions only, never the browser.
- The browser holds only the **Supabase anon key** (publishable, fine to ship as `VITE_SUPABASE_ANON_KEY`) and the user's session JWT.

## Quick checklist

- [ ] Google Cloud OAuth client in a **Workspace-owned** project, consent screen **User Type = Internal**.
- [ ] Supabase Google provider configured; redirect URLs allow-listed.
- [ ] `signInWithOAuth` passes `hd: 'alfadocs.com'` (hint) — and you do **not** trust it for security.
- [ ] Server-side domain enforcement (trigger or Auth Hook) rejects non-`@alfadocs.com`.
- [ ] RLS deny-by-default; policies check the email/role claim.
- [ ] React route guard via `getSession` + `onAuthStateChange`.
- [ ] Sign-in UI is the kit's `SocialSignInButton`; no client secret in the frontend.
