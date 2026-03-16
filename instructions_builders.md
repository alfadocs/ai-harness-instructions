## Using Lovable with the AI Harness (Non‑technical Guide)

This guide is for **non‑developers** who use Lovable (or similar AI code tools) to build apps that will run on the **AI Harness pipeline** you’ve just read about.

The goal: if you follow these guidelines when you create or edit a project in Lovable, the guardrails, deployment, and Supabase automation in this repo will “just work” with minimal manual fixes.

---

## 1. What you are building (mental model)

When you ask Lovable to build an app for this system, you are *not* building a standalone frontend. You are always building:

- A **web UI** (React/Vite etc.)
- Backed by a **Backend‑for‑Frontend (BFF)**, usually implemented as:
  - Supabase Edge Functions, or
  - Node server routes in the same repo (which the pipeline can still deploy)

The important part:

- The **frontend must talk only to the BFF**, not directly to AlfaDocs APIs or Supabase.
- The **BFF** is the single gateway that:
  - Handles **authentication with AlfaDocs**.
  - Talks to **Supabase** (database, storage, etc.).

You don’t have to write code by hand, but you do need to tell Lovable the right things.

---

## 2. What to tell Lovable (the prompt)

You don’t need to invent a perfect prompt yourself. We already have one for you here:

- `https://raw.githubusercontent.com/alfadocs/ai-harness-instructions/refs/heads/main/foundational-prompt.md`

This “foundational prompt” tells Lovable to:

- Build your app in the correct **Backend‑for‑Frontend (BFF)** style.
- Always implement **AlfaDocs login and session handling first**.
- Keep all **secrets and API keys on the backend**, never in the browser.
- Make the frontend call only your **own backend/API**, not AlfaDocs or Supabase directly.

### How to use it in Lovable

1. When creating a new project (or a big new feature), **open the foundational prompt URL** above.
2. **Copy the entire text** from that page.
3. In Lovable’s prompt box, **paste that text first**.
4. Under it, in your own words, add what you want the app to do, for example:
   - “This app should let an AlfaDocs user see a list of patients and search them by name.”
5. When you later ask Lovable for changes, keep reminding it:
   - “Remember: use the same BFF/auth pattern from the Alfadocs foundational prompt.”

If you always start from that foundational prompt, the generated project will naturally follow the rules this pipeline expects (auth first, BFF in front of AlfaDocs + Supabase, secrets on the backend).

---

## 3. How auth should work (in plain language)

You don’t need to know all the code, but you should recognize the **shape** of the flow Lovable produces:

- **Login**
  - User clicks “Login with AlfaDocs”.
  - Browser is redirected to AlfaDocs to sign in.
  - After sign‑in, AlfaDocs redirects back to your app’s **callback URL**.
  - The BFF (Supabase function / Node route) exchanges the temporary code for real tokens.
  - The BFF stores those tokens securely (e.g. in Supabase DB) and creates a **session ID**.
  - The session ID is sent to the browser in a **secure, HttpOnly cookie**.

- **Using the app**
  - The browser makes requests to your BFF endpoints (e.g. `/api/...` or Supabase functions).
  - The cookie is sent automatically; the browser never sees raw tokens.
  - The BFF looks up the session, uses the stored AlfaDocs/Supabase credentials, and performs actions.

- **Logout**
  - The BFF clears the session (and cookie) and optionally revokes tokens.

If you see Lovable generating code that:

- Calls AlfaDocs APIs **directly from the browser**, or
- Stores tokens in `localStorage` or exposes secrets in frontend code,

