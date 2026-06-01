---
name: alfadocs-patient-deep-link
description: Use when your app needs to jump or deep-link into AlfaDocs — e.g. an "Open in AlfaDocs" button, opening a patient's record/chart in AlfaDocs, linking a row back to the patient's show page, or navigating the user into a specific practice in AlfaDocs.
---

# AlfaDocs patient deep link

A deep link sends the user from your Lovable app straight to a patient's record inside AlfaDocs (`app.alfadocs.com`), in the right practice, with the right archive already selected. Use it for "Open in AlfaDocs" buttons, "View full record" links, or any hand-off where the user should continue their work in AlfaDocs itself.

This is a **navigation** pattern (a browser URL), not an API call. You do not need an access token to build the link — AlfaDocs handles authentication when the user lands on it. You only need the patient's numeric IDs (which you get from the AlfaDocs API).

## The URL pattern

```
https://app.alfadocs.com/login?redirectPracticeId={practiceId}&redirect=%2FdoctorPatient%2Fshow%2F{patientId}
```

- `{practiceId}` — integer ID of the practice the patient belongs to.
- `{patientId}` — integer ID of the patient.

When the user clicks this:

1. AlfaDocs ensures the user is signed in (shows its own login if needed — your app never handles their AlfaDocs password).
2. It logs them into the practice given by `redirectPracticeId`, which sets the practice's default **archive** in the session automatically.
3. It forwards them to the `redirect` target — the patient's show page.

You do **not** put the archive ID in the link. The practice login step selects the default archive for you.

## Where the IDs come from

You need `practiceId` and `patientId`. Get them from the AlfaDocs API while authenticated (OAuth via your Supabase Edge Function BFF — see your AlfaDocs API skill).

- `practiceId` is the practice the patient lives in. If your app is scoped to one practice, you already have it. Otherwise read it from `GET /me` (returns `practiceId` and `archiveId`) or from the patient object you fetched.
- `patientId` is the `id` of the patient record returned by the patients endpoint, e.g. `GET /practices/{practiceId}/archives/{archiveId}/patients`.

API base: `https://app.alfadocs.com/api/v1`. Reference: <https://app.alfadocs.com/api.html>.

> The IDs are not secret, but the access token you used to fetch them **is** — keep all token handling in the Supabase Edge Function, never in the frontend.

## Building the link safely

The `redirect` value is a path that contains slashes, so it **must be URL-encoded** before it goes in the query string (`/` becomes `%2F`). Do not hand-concatenate the encoded string — build the path normally and let the platform encode it.

```ts
function alfadocsPatientLink(practiceId: number, patientId: number): string {
  const url = new URL("https://app.alfadocs.com/login");
  url.searchParams.set("redirectPracticeId", String(practiceId));
  // set() encodes the value for you — pass the raw path, not a pre-escaped one
  url.searchParams.set("redirect", `/doctorPatient/show/${patientId}`);
  return url.toString();
}
```

Render it as a normal link that opens in a new tab so the user keeps your app open:

```tsx
<a
  href={alfadocsPatientLink(practiceId, patientId)}
  target="_blank"
  rel="noopener noreferrer"
>
  Open in AlfaDocs
</a>
```

## Linking to a specific tab

To land on a tab of the patient page (for example appointments), add a fragment to the path. `URL.searchParams.set` encodes the `#` for you when you pass the raw value:

```ts
url.searchParams.set("redirect", `/doctorPatient/show/${patientId}#appointments`);
// → ...&redirect=%2FdoctorPatient%2Fshow%2F5678%23appointments
```

## Example

Practice `1234`, patient `5678`:

```
https://app.alfadocs.com/login?redirectPracticeId=1234&redirect=%2FdoctorPatient%2Fshow%2F5678
```

## If the user is already in AlfaDocs

When you know the user already has an active AlfaDocs session, you can skip the `/login` step and go straight to the practice login. Encoding is optional here but still safe to apply:

```
https://app.alfadocs.com/doctorAddress/login/{practiceId}?redirect=/doctorPatient/show/{patientId}
```

For most builders the standard `/login?redirectPracticeId=...` form is the right default — it works whether or not the user is signed in.

## Gotchas

- **Always encode `redirect`.** An unencoded `/doctorPatient/show/5678` will be cut off at the first `/` and the link breaks. Use `URL` / `searchParams.set` rather than string templates.
- **Don't double-encode.** If you build with `searchParams.set`, pass the raw path — do not pass `%2F...`, or you'll get `%252F` and a broken link.
- **No archive ID in the link.** Adding it does nothing; the practice login picks the default archive.
- **Wrong practice = patient not found.** `practiceId` must be the practice that owns `patientId`. If you support multiple practices, carry the practice ID alongside each patient when you fetch them.
- **This is a link, not an API endpoint.** Don't send an `Authorization` header to `/login` — it's a normal browser navigation handled by AlfaDocs.
- **Open in a new tab** (`target="_blank"` + `rel="noopener noreferrer"`) so your app stays open behind it.
