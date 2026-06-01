# Reference Edge Function (API-key proxy)

A single Supabase Edge Function that keeps `ALFADOCS_API_KEY` server-side, resolves `practiceId`/`archiveId` from `/me`, and forwards a requested AlfaDocs path. The browser calls this function; it never sees the key.

## How the frontend calls it

Send a relative AlfaDocs path (after `/practices/{practiceId}/archives/{archiveId}`) plus the HTTP method. The function fills in the IDs and headers.

```ts
// frontend
const res = await fetch(`${SUPABASE_FUNCTIONS_URL}/alfadocs-proxy`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ method: 'GET', path: '/patients' }),
});
const patients = await res.json();
```

## The function

```ts
// supabase/functions/alfadocs-proxy/index.ts
import { serve } from 'https://deno.land/std/http/server.ts';

const BASE = 'https://app.alfadocs.com/api/v1';

// Cached per cold start so we don't hit /me on every request.
let ctx: { practiceId: string; archiveId: string } | null = null;

function alfaHeaders() {
  const key = Deno.env.get('ALFADOCS_API_KEY');
  if (!key) throw new Error('ALFADOCS_API_KEY is not set');
  return { 'X-Api-Key': key, Accept: 'application/json' };
}

async function getContext() {
  if (ctx) return ctx;
  const res = await fetch(`${BASE}/me`, { headers: alfaHeaders() });
  if (!res.ok) throw new Error(`/me failed: ${res.status}`);
  const me = await res.json();
  ctx = { practiceId: String(me.practiceId), archiveId: String(me.archiveId) };
  return ctx;
}

serve(async (req) => {
  try {
    const { method = 'GET', path, body } = await req.json();
    const { practiceId, archiveId } = await getContext();

    // The frontend never picks practiceId/archiveId — they come from /me.
    const url = `${BASE}/practices/${practiceId}/archives/${archiveId}${path}`;

    const upstream = await fetch(url, {
      method,
      headers: {
        ...alfaHeaders(),
        ...(body ? { 'Content-Type': 'application/json' } : {}),
      },
      body: body ? JSON.stringify(body) : undefined,
    });

    const text = await upstream.text();
    return new Response(text, {
      status: upstream.status,
      headers: { 'Content-Type': 'application/json' },
    });
  } catch (e) {
    return new Response(JSON.stringify({ error: String(e) }), { status: 500 });
  }
});
```

## Notes

- **Never** return the key or echo it in error messages.
- Validate/whitelist `path` and `method` if the frontend is untrusted, so it can only reach the endpoints your app actually uses.
- If you need an endpoint outside the `/practices/{practiceId}/archives/{archiveId}` tree, branch on `path` and build the URL accordingly — but always source the IDs from `/me`, never from the request body.
- Verify every endpoint shape against https://app.alfadocs.com/api.html before wiring it up.
