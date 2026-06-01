---
name: alfadocs-gdpr-consent
description: Use when building or changing any AlfaDocs-connected feature that stores, transfers, or processes patient or personal data — adding a consent checkbox, sending patient data to an external service, logging consent, or anything touching GDPR/privacy.
---

# GDPR & explicit consent for AlfaDocs connected apps

Any feature that moves patient or personal data **outside the user's local browser session** — sending it to an external API, a third-party service (Google, Make, email, AI), or storing it in your Supabase database — MUST capture explicit, logged consent first. This is non-negotiable for AlfaDocs apps handling dental-practice patient data.

"Personal data" here means anything that identifies a patient: name, date of birth, contact details, fiscal code / tax ID, clinical notes, appointment history, documents, etc.

## The core rule

> No outgoing data action until the user has explicitly consented, and that consent is recorded with who + when.

## Explicit consent checkbox — the pattern

Implement a checkbox the user must actively tick before any data leaves the local session.

- **Place it on every page or section** where an action can send data outside the local client. Don't rely on a single global setting buried elsewhere.
- **Disable every trigger** (Send / Export / Sync / Submit buttons) until the checkbox is ticked. The control should be visibly disabled, not just rejected on click.
- **Never pre-tick it.** Consent must be an affirmative, opt-in action by the user. A default-checked box is not valid consent under GDPR.
- **Use clear, specific wording** — say what data is sent, where, and why. Avoid vague "I agree to the terms".
- **Keep the checkbox visible after consent** so the user can revoke it by unticking. Revoking stops *future* sends; it must NOT delete prior consent records (you need them as proof).

Example checkbox label:

```
I understand that ticking this box sends this patient's personal data to
<destination> for the purpose of <purpose>, and I confirm I have a lawful
basis to do so.
```

## Logging consent — store proof

Every time consent is given, write a record to your Supabase database. Store enough to prove consent was given by a specific user at a specific time.

Recommended columns for a `consent_log` table:

| Column | Purpose |
| --- | --- |
| `id` | primary key |
| `user_id` | the authenticated practice user who consented |
| `practice_id` / `archive_id` | AlfaDocs scope (from `/me`) the consent applies to |
| `patient_ref` | which patient/record the consent covers (if per-patient) |
| `action` | what the consent authorises (e.g. `export_to_google`) |
| `consented_at` | timestamp (UTC) |
| `fingerprint` | a stable identifier for the consenting session/user |
| `revoked_at` | nullable; set when the user unticks, never delete the row |

Write rules:

- **Append-only.** Revoking sets `revoked_at`; it never deletes the original consent row.
- **Write the log from a Supabase Edge Function**, server-side, so the timestamp and user identity are trustworthy and can't be forged from the frontend.
- **Check for a valid (non-revoked) consent record before every outgoing data call** — don't trust only the checkbox state in the browser.

## The practice's legal responsibilities

The dental practice (your app's user) — not the integration builder, not AlfaDocs — is the data controller. Make sure the app supports them in meeting these obligations, and surface them clearly in the consent UI / docs:

- A proper **legal basis** for processing patients' personal data.
- **Informing patients** that their data is processed via this integration.
- **Appropriate security** during transfer and storage (HTTPS everywhere, secrets server-side — see below).
- **Data minimisation & purpose limitation** — send only the fields the feature actually needs, and only for the stated purpose.
- **Proper agreements** with every third party in the data path (AlfaDocs, Google, Make, your AI provider, etc.).

## Security baseline (don't undermine consent with leaks)

- **Secrets stay server-side.** OAuth client secret, API keys, and third-party credentials live in **Supabase Edge Function secrets** — never in the frontend bundle and never in a `.env` committed to git.
- **All patient-data calls (to the AlfaDocs API at `https://app.alfadocs.com/api/v1` and to any third party) go through a Supabase Edge Function**, not directly from the browser. This keeps tokens off the client and lets you enforce the consent check server-side.
- Scope every call to the authenticated practice — fetch `practiceId` + `archiveId` from `/me` and store consent against them.

## Before you ship

- **Never connect a real practice to a prototype.** During development connect only test practices with fake patient data.
- **Consult a data-protection expert or legal adviser** before going to production with real patient data.
- Confirm the disable-until-consent behaviour, the append-only consent log, and the server-side consent check are all in place.

## Quick checklist

- [ ] Consent checkbox on every page that sends data outside the local client
- [ ] Not pre-ticked; affirmative opt-in only
- [ ] Outgoing-action buttons disabled until ticked
- [ ] Consent logged (user + practice + timestamp + fingerprint) to Supabase, append-only
- [ ] Server-side consent check before every outgoing data call
- [ ] Revoke = set `revoked_at`, never delete prior records
- [ ] Secrets in Supabase Edge Function secrets; data calls via Edge Function
- [ ] Only test practices connected until legally reviewed
