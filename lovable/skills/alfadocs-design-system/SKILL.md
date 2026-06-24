---
name: alfadocs-design-system
description: Use when building or styling ANY UI, page, screen, or component for an AlfaDocs app — install, wire, and use the @alfadocs/ui-kit React design system instead of Material UI, Bootstrap, shadcn, Chakra, or hand-rolled CSS.
---

# AlfaDocs design system (@alfadocs/ui-kit)

`@alfadocs/ui-kit` is the AlfaDocs design system: brand tokens, four themes, ~175 pre-styled React components, and i18n. **Use it for every piece of UI.** Do not introduce another component library or write your own styles.

## The one rule that matters most

> **Kit-first, always. If the kit has a component for what you need, you MUST use it — inventing or hand-rolling one the kit already ships is a defect, not a shortcut.** Never reach for Material UI, Bootstrap, shadcn/ui, Chakra, or hand-rolled CSS/markup.

This is the rule agents break most often, **even with a template and these skills loaded**: they reinvent a primitive the kit already has — a coloured `<div>` "badge", a hand-built `<table>`, a bespoke modal, a custom avatar circle, a styled `<button>` — instead of importing the real one. Don't. The kit ships **~175 components**; the thing you're about to build almost certainly already exists.

**The mandatory workflow for every piece of UI — no exceptions:**

1. **Look it up first.** Before writing a component, find the kit primitive for it: the catalogue at **[storybook.alfadocs.cloud](https://storybook.alfadocs.cloud)** (source of truth for every component, variant, state, prop) and **[component-selection.md](./component-selection.md)** (maps "the thing I want to build" → the kit component). Assume one exists.
2. **Use it as-is.** Import the real component and drive it through its props/variants. Do NOT re-implement its appearance with raw Tailwind/`<div>`s, and do not wrap your own styling around a bare HTML element to imitate it.
3. **Only if it genuinely doesn't exist:** STOP and ask the builder. Do not hand-roll it, do not pull in another library, do not approximate it with styled `<div>`s.

**Commonly reinvented — these all exist; never hand-roll them:** `Button`, `IconButton`, `Badge`, `Tag`, `Card`, `Dialog`/`Sheet`, `DataTable`, `Tabs`, `Avatar` (every person — see below), `EmptyState`, `Tooltip`, `Select`/`Combobox`/`MultiSelect`, `Switch`, `Checkbox`, `Radio`, `Spinner`, `Skeleton`, `Stat`, `Pagination`, `Breadcrumb`, `Toast` (via `Toaster`), and the whole app chrome (`MarketplaceAppShell`). When in doubt, search the catalogue — don't assume it's missing.

## 1. Install

The kit is on the **public npm registry**:

```bash
npm install @alfadocs/ui-kit
```

- **React 18 or 19** peer (`^18.2.0 || ^19.0.0`) + `react-dom`. React 19 support landed in kit **0.36.0**; on older kit versions, install React-19 apps with `--legacy-peer-deps`.
- Also needs `i18next` and `react-i18next` (peers).
- No Tailwind required — components ship **pre-styled**. You only import the tokens.

If an install fails with `ETARGET`, an old `.npmrc` is pointing the scope at the wrong registry. Remove any `@alfadocs:registry=...` line so it resolves from public npm.

## 2. Import the tokens once, at the app root

```ts
import '@alfadocs/ui-kit/tokens';
```

This defines every CSS custom property (colours, spacing, type) and the in-app font. **Without it, components render unstyled.** Import it exactly once, at the top of your app entry.

## 3. Wrap your app in the providers

```tsx
import { ThemeRoot, TooltipProvider, Toaster } from '@alfadocs/ui-kit';
import { I18nextProvider } from 'react-i18next';
import i18n from './i18n'; // your own i18next instance — see i18n section

<I18nextProvider i18n={i18n}>
  <ThemeRoot theme="light" accessible={false}>
    <TooltipProvider delayDuration={700} skipDelayDuration={300}>
      <App />
      <Toaster />
    </TooltipProvider>
  </ThemeRoot>
</I18nextProvider>;
```

- **`ThemeRoot`** applies the active theme.
- **`TooltipProvider`** is required for any tooltip-bearing component.
- **`Toaster`** is the toast render target (place it once, inside the tree).

## 4. Use components — prefer per-component imports

Per-component subpath imports tree-shake cleanly and avoid pulling in heavy optional dependencies:

```tsx
import { Button } from '@alfadocs/ui-kit/button';
import { Dialog } from '@alfadocs/ui-kit/dialog';

<Button variant="primary">Save</Button>;
```

The root barrel (`import { Button } from '@alfadocs/ui-kit'`) works too, but only tree-shakes safely when every optional peer is installed — so subpaths are the safer default.

A few heavy components need an extra optional peer (Calendar → `@fullcalendar/*`, DataTable → `ag-grid-react`, Chart → `apexcharts`, RichTextEditor → `@tiptap/*`, PaymentForm → `@stripe/*`, etc.). See [components-and-rules.md](./components-and-rules.md) for the full table.

### Pick the *right* component — don't hand-roll

The kit has a **dedicated** primitive for most cases — `PhoneInput`, `OtpInput`,
`EmailInput`, `DataTable`, `AlertDialog` (destructive confirms), `EmptyState`,
`ContactCard`, `SignInWithAlfadocsButton` (the AlfaDocs login CTA), and more.
Reaching for a generic `TextInput` / `Button` / `Dialog` when a specialised one
exists is the most common mistake. **[component-selection.md](./component-selection.md)**
maps each legacy pattern → the right component, with an anti-patterns checklist.

### People & names — always go through `Avatar`

Whenever the UI shows a **person** — a patient, operator, staff member, the signed-in
user, a list row, a comment author, anyone — render the kit **`Avatar`** for the visual
identity. Never hand-roll a coloured circle, hand-write initials, or drop in a raw
`<img>` for a person. `Avatar` deterministically derives the initials and a stable brand
colour from the `name` you pass, and shows the photo when `src` is set:

```tsx
import { Avatar } from '@alfadocs/ui-kit/avatar';
<Avatar name={person.fullName} src={person.photoUrl} size="md" />
```

And present the name **one way, everywhere**. Pick a single field (e.g. one `fullName`
string) and reuse it — pass that same string to `Avatar`'s `name` prop *and* render that
same string as the visible label. Don't mix `"John Doe"`, `"J. Doe"`, and `"Doe, John"`
across a screen — or even within the same view. For a fuller person block (name + role /
contact details) use `ContactCard` / `contact-profile-card` rather than re-assembling an
avatar + text by hand.

### Build the app frame from the shell — never hand-roll the chrome

**Every AlfaDocs app's chrome — header, sidebar, account menu, and the pre-auth login — comes from ONE kit pattern. Do NOT build your own header / sidebar / top-bar / login screen, and never wrap the app in a shadcn, Material, or Bootstrap layout.** This is the single most common thing apps get wrong. Browse it first in Storybook: **`Patterns/Public/MarketplaceAppShell`** ([storybook.alfadocs.cloud](https://storybook.alfadocs.cloud)).

```tsx
import { MarketplaceAppShell, ConnectWithAlfadocs } from '@alfadocs/ui-kit/patterns/marketplace-app-shell';
```

- **`MarketplaceAppShell`** — the branded app frame: header brand lockup, sidebar nav, account menu (avatar + sign-out). Key props: `productName`, `nav` (array of `{ id, label, href, icon?, badgeCount?, isActive? }`), `user` (`{ name, email?, avatarSrc? }`), `labels`, `onSignOut`, and `renderLink` wired to your router's `<Link>`. `variant`: `'standalone'` (default — full header + sidebar) or `'embedded'` (for running INSIDE the AlfaDocs platform iframe — drops the header/sidebar/lockup and renders `nav` as a slim top tab strip).
- **`ConnectWithAlfadocs`** — the pre-auth "Connect with AlfaDocs" screen (`status`: `idle`/`connecting`/`error`; `layout`: `'card'`/`'split'`).

Auth is **decoupled** — the kit never touches OAuth or secrets. Wire `user`/`onSignOut` and `status`/`onConnect` to your session (e.g. `useAlfadocsAuth` from `@alfadocs/auth/react` + your server-side BFF — see the `alfadocs-connected-app-oauth` skill).

#### The brand lockup is "{productName} by Alfadocs" — leave it as-is

The shell brands your app with the kit's **`ProductLockup`** in its **`maker`** variant: it renders **"{productName} by Alfadocs"** — your app name leads, followed by the real Alfadocs wordmark. This is the **correct, intended branding for a marketplace / connected app** (kit **0.52.0+**), and `MarketplaceAppShell` + `ConnectWithAlfadocs` apply it **automatically**. So:

- Just pass your app's name as `productName` and **leave the lockup alone**. Don't rewrite it to "Alfadocs {name}", don't swap in your own logo, and don't restyle the header brand.
- `ProductLockup`'s other variant — `standard` ("Alfadocs {name}") — is for first-party **internal** AlfaDocs products only. Those are not built on this shell, so you will not use it here.

## 5. Themes

One token set, four themes, set via `ThemeRoot` (or a class on the root element):

| Theme | `ThemeRoot` props | Class(es) |
|---|---|---|
| Light (default) | `theme="light" accessible={false}` | `theme-light` |
| Dark | `theme="dark" accessible={false}` | `theme-dark` |
| Light, accessible (AAA) | `theme="light" accessible` | `theme-light theme-accessible` |
| Dark, accessible (AAA) | `theme="dark" accessible` | `theme-dark theme-accessible` |

Read/change the theme at runtime with the `useTheme()` hook → `{ theme, accessibilityMode, setTheme, setAccessibilityMode }`. Themes also honour `prefers-color-scheme`, `prefers-contrast`, `prefers-reduced-motion`, and `forced-colors`.

## 6. Translations (i18n)

The kit reads its own strings from the **`ui`** namespace via `react-i18next`. Your app owns the i18next instance.

```ts
// quickest path — auto-registers the kit's bundles (en/it/de/ar)
import '@alfadocs/ui-kit/i18n';
```

- Keep **your** app strings under a separate `app` namespace; never put app strings in `ui.*`.
- All user-visible text must go through `useTranslation()` — no hardcoded strings.
- 18 locales supported; **en / it / de** are the production-ready set today; others fall back to English.
- RTL (`ar`) works automatically because the kit uses CSS logical properties. CJK fonts (`zh`/`ja`) are **not** bundled — add the Noto Sans SC/JP stylesheet to your `<head>` if you support them.

## Consumer rules (non-negotiable)

These keep your UI on-brand, accessible, and theme/RTL-safe. Full detail in [components-and-rules.md](./components-and-rules.md). The short version:

- **Kit-first — never invent** — if the kit ships a component for it, use that component; hand-rolling or approximating one it already has (a `<div>` badge, a hand-built table, a custom modal, an avatar circle) is a defect. If something is genuinely missing, STOP and ask — don't pull in another UI library or fake it with styled `<div>`s.
- **People go through `Avatar`** — every person/name renders the kit `Avatar` (consistent initials + deterministic brand colour + photo); never hand-roll a circle, hand-write initials, or drop in a raw `<img>` for a person. Use ONE name format everywhere — the same `fullName` string for `Avatar`'s `name` prop and the visible label.
- **Tokens, not literals** — every colour/spacing/radius/shadow/font uses a `var(--…)` token, never raw hex/rgb/hsl.
- **NEVER invent token names.** The kit's real semantic tokens are `--background`, `--foreground`, `--card`, `--card-foreground`, `--primary`, `--primary-foreground`, `--muted`, `--muted-foreground`, `--border`, `--input`, `--ring`, `--destructive`, `--destructive-foreground`, `--accent`, `--accent-foreground`, `--success`, `--warning`, `--error`, `--info`. Aliases like `--color-background`, `--color-danger`, `--color-text-muted`, `--ds-font-sans` **do not exist** — they silently fall back to hardcoded hex, breaking all four themes. When unsure, check [storybook.alfadocs.cloud](https://storybook.alfadocs.cloud) or `src/tokens/index.css`.
- **No inline `style={{}}`** — use Tailwind utility classes referencing `var(--…)` tokens (e.g. `className="bg-[var(--background)] text-[var(--foreground)]"`). Inline styles bypass the theme system entirely.
- **Delete `src/components/ui/`** — Lovable scaffolds a shadcn/ui boilerplate folder by default; delete the whole directory. Import every component from `@alfadocs/ui-kit` instead. Never import from `@/components/ui/`.
- **Delete the scaffolded Tailwind colour block in `src/styles.css`** — Lovable generates a `:root { --background: oklch(…); --foreground: oklch(…); … }` block with raw colour values. Delete it entirely. `ThemeRoot` (from `@alfadocs/ui-kit`) owns those tokens via its `theme-light`/`theme-dark` class; the `:root` block overrides or conflicts with the kit's palette. CSS aliases like `--color-background: var(--background)` are fine; raw `oklch(…)`/`hsl(…)`/`hex` values at `:root` are not.
- **App chrome comes from the shell** — the header, sidebar, account menu, and login screen are `MarketplaceAppShell` / `ConnectWithAlfadocs` (`@alfadocs/ui-kit/patterns/marketplace-app-shell`), never hand-rolled or a shadcn/Material layout. Leave the shell's "{productName} by Alfadocs" brand lockup as-is — don't override it or drop in your own logo.
- **Logical properties only** — `margin-inline-start/end`, `padding-inline-start/end`, `text-start/end` (Tailwind `ms-*`/`me-*`/`ps-*`/`pe-*`). Never `margin-left`, `text-right`, `ml-*`, etc.
- **All strings via `useTranslation()`** — kit strings `ui.*`, your strings `app.*`.
- **In-app font is `--font-sans`** (Space Grotesk). `--font-serif` (Fraunces) is marketing-only — keep it out of app UI.
- **Raw HTML only via `SafeHtml`** (`@alfadocs/ui-kit/safe-html`) — never `dangerouslySetInnerHTML` directly.
- **Don't fight the overlay system** — never register a capture-phase `document` `pointerdown` listener that calls `stopPropagation()`; it breaks every dismissable primitive (dialogs, popovers, dropdowns).
- **No other UI libraries** — no Material UI, Bootstrap, shadcn/ui, Chakra, or hand-rolled CSS.

## Secrets reminder

If your UI talks to the AlfaDocs API or OAuth, API keys and the OAuth client secret live in **Supabase Edge Function secrets** — never in the frontend or a committed `.env`.

## Next

- [Which component for which job](./component-selection.md) — pick the right primitive
- [Component catalogue + full consumer rules](./components-and-rules.md)
- Live Storybook: **[storybook.alfadocs.cloud](https://storybook.alfadocs.cloud)**
