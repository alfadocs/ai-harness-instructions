---
name: alfadocs-webhooks
description: Use when receiving, parsing, or handling AlfaDocs webhooks / event notifications in a Supabase Edge Function — e.g. AlfaDocs POSTs an event like appointment_created or patient_created to your app and you need to read it, route on archiveId, and respond correctly.
---

# Receiving AlfaDocs webhooks

AlfaDocs notifies your app of events (appointments, patients, documents, etc.) by POSTing to a webhook URL. Get three things right or the integration silently breaks:

1. The body is **form-urlencoded, not JSON**, in **indexed bracket notation**.
2. **One POST can carry many events** (indices `0`, `1`, `2`, …).
3. You **must respond HTTP 200** — never 202 (AlfaDocs treats 202 as an error and retries / alerts).

Receive webhooks in a **Supabase Edge Function** (a public endpoint AlfaDocs can reach). Give AlfaDocs the function's URL to register as the webhook target.

---

## 1. The payload format

AlfaDocs sends `Content-Type: application/x-www-form-urlencoded`. Do **not** call `await req.json()` — it throws. Read the raw body and parse it as form data.

The fields use indexed bracket keys. A single event arrives like this:

```
0[event]=appointment_created
0[objectId]=12345
0[data][created_entity][id]=12345
0[data][created_entity][archiveId]=67890
```

Multiple events in one call just keep incrementing the leading index:

```
0[event]=appointment_created
0[objectId]=12345
0[data][created_entity][archiveId]=67890
1[event]=patient_updated
1[objectId]=98765
1[data][updated_entity][archiveId]=67890
```

Your job is to turn those flat bracket keys into structured objects:

```js
[
  { event: "appointment_created", objectId: "12345",
    data: { created_entity: { id: "12345", archiveId: "67890" } } },
  { event: "patient_updated", objectId: "98765",
    data: { updated_entity: { archiveId: "67890" } } },
]
```

---

## 2. Parsing — read raw body, expand bracket keys, group by index

Use `URLSearchParams` to read the pairs, then walk each key like `0[data][created_entity][archiveId]` into a nested object grouped by the leading index.

```ts
// supabase/functions/alfadocs-webhook/index.ts
import { serve } from "https://deno.land/std/http/server.ts";

interface AlfaDocsEvent {
  event: string;
  objectId?: string;
  data: Record<string, unknown>;
}

function groupFormDataToEvents(params: URLSearchParams): AlfaDocsEvent[] {
  const byIndex: Record<string, Record<string, unknown>> = {};

  for (const [rawKey, value] of params.entries()) {
    // rawKey looks like: 0[data][created_entity][archiveId]
    const match = rawKey.match(/^(\d+)(.*)$/);
    if (!match) continue;

    const [, index, rest] = match;          // index = "0", rest = "[data][created_entity][archiveId]"
    const path = [...rest.matchAll(/\[([^\]]+)\]/g)].map((m) => m[1]); // ["data","created_entity","archiveId"]

    byIndex[index] ??= {};
    let node: Record<string, unknown> = byIndex[index];
    path.forEach((segment, i) => {
      if (i === path.length - 1) {
        node[segment] = value;
      } else {
        node[segment] ??= {};
        node = node[segment] as Record<string, unknown>;
      }
    });
  }

  // Return events ordered 0,1,2,… as structured objects
  return Object.keys(byIndex)
    .sort((a, b) => Number(a) - Number(b))
    .map((i) => byIndex[i] as unknown as AlfaDocsEvent);
}

serve(async (req) => {
  if (req.method !== "POST") {
    return new Response("Method not allowed", { status: 405 });
  }

  // Read the RAW body — it is form-urlencoded, NOT JSON.
  const rawBody = await req.text();
  const events = groupFormDataToEvents(new URLSearchParams(rawBody));

  for (const evt of events) {
    await handleEvent(evt); // your routing — see below
  }

  // ALWAYS 200. Never 202.
  return new Response("OK", { status: 200 });
});
```

---

## 3. Find the practice via `archiveId` from the payload

The webhook does **not** hand you a `practiceId`. It gives you an `archiveId` nested inside the entity data. Look up which practice that archive belongs to in your own Supabase table (the one you populated when the practice connected — e.g. by calling `/me` during OAuth and storing its `archiveId` + `practiceId`).

```ts
async function handleEvent(evt: AlfaDocsEvent) {
  // The entity key varies by event: created_entity / updated_entity / deleted_entity
  const entity =
    (evt.data.created_entity ??
      evt.data.updated_entity ??
      evt.data.deleted_entity) as { id?: string; archiveId?: string } | undefined;

  const archiveId = entity?.archiveId;
  if (!archiveId) return; // nothing routable; still respond 200 overall

  // Map archiveId -> the connected practice (use the Supabase service role inside the function)
  const { data: practice } = await supabase
    .from("connected_practices")
    .select("practice_id, archive_id")
    .eq("archive_id", archiveId)
    .maybeSingle();

  if (!practice) return; // unknown archive — log it, but still respond 200

  // ... do your work scoped to practice.practice_id (this is your tenant key) ...
}
```

Key points:
- `archiveId` is the routing key in the payload; `practiceId` is what you resolve it to and what you tenant-scope everything by afterwards.
- Run all DB access inside the Edge Function with the **service role**; the webhook request is unauthenticated by Supabase's RLS, so never expose these tables to `anon`/`authenticated`.
- An unknown or missing `archiveId` is not a reason to fail — log and skip that one event, still return 200 for the call.

---

## 4. Always respond HTTP 200 — never 202

This is the single most common mistake. AlfaDocs interprets a **202** (or any non-200) as a delivery failure and will treat the webhook as errored.

- Return `200` once you have **received and parsed** the payload — you do not have to finish all downstream work first.
- Do **not** use `202 Accepted`, even though "Accepted, processing later" feels semantically right. AlfaDocs does not.
- Wrap your handling in try/catch so a bug while processing one event still lets you return 200 for the call. Log the failure on your side and reconcile separately; don't surface it as a non-200 to AlfaDocs.

```ts
serve(async (req) => {
  try {
    const events = groupFormDataToEvents(new URLSearchParams(await req.text()));
    for (const evt of events) {
      try { await handleEvent(evt); }
      catch (e) { console.error("event handling failed", evt.event, e); }
    }
  } catch (e) {
    console.error("webhook parse failed", e);
  }
  return new Response("OK", { status: 200 }); // 200, always
});
```

---

## Gotchas checklist

- [ ] Read `await req.text()` — never `await req.json()` (the body is form-urlencoded).
- [ ] Parse the indexed bracket keys; expect indices `0,1,2,…` for multiple events in one POST.
- [ ] Route on `archiveId` from inside `data.*_entity`, then resolve it to your stored `practiceId`.
- [ ] Respond **200**. Never 202. Never let a processing error bubble up as a non-200.
- [ ] All DB work inside the function uses the Supabase **service role**; webhook tables stay locked to `anon`/`authenticated`.
- [ ] Any API keys live in **Supabase Edge Function secrets**, never in the frontend or a committed `.env`. (AlfaDocs webhooks are **not** HTTP-signed — there's no signature/shared secret to verify; don't add one.)

## Reference

- AlfaDocs API base URL: `https://app.alfadocs.com/api/v1` — call `/me` (while authenticated) to get the `practiceId` + `archiveId` you store and match webhooks against.
- API docs: https://app.alfadocs.com/api.html
