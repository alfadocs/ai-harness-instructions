---
name: alfadocs-design-system
description: Use when building or styling ANY UI, page, screen, or component for an AlfaDocs app — install, wire, and use the @alfadocs/ui-kit React design system instead of Material UI, Bootstrap, shadcn, Chakra, or hand-rolled CSS.
---

# AlfaDocs design system (@alfadocs/ui-kit)

`@alfadocs/ui-kit` is the AlfaDocs design system: brand tokens, four themes, ~125 pre-styled React components, and i18n. **Use it for every piece of UI.** Do not introduce another component library or write your own styles.

## The one rule that matters most

> Build the UI out of `@alfadocs/ui-kit` components. If the kit has a component for what you need (button, input, dialog, card, table…), use it. Never reach for Material UI, Bootstrap, shadcn/ui, Chakra, or hand-rolled CSS.

Before building anything, browse the live catalogue so you know what already exists: **[storybook.alfadocs.com](https://storybook.alfadocs.com)** is the source of truth for every component, variant, state, and prop.

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

### Building a marketplace app? Use the shell

The kit ships the whole app frame and the pre-auth screen as one pattern (kit **0.36+**):

- **`MarketplaceAppShell`** — branded AppFrame: header with the AlfaDocs product lockup, sidebar nav, account menu (avatar + sign-out).
- **`ConnectWithAlfadocs`** — the pre-auth "Connect with AlfaDocs" screen (`status`: `idle`/`connecting`/`error`).

```tsx
import { MarketplaceAppShell, ConnectWithAlfadocs } from '@alfadocs/ui-kit/patterns/marketplace-app-shell';
```

Auth is **decoupled** — the kit never touches OAuth or secrets. Wire `user`/`onSignOut` and `status`/`onConnect` to your session (e.g. `useAlfadocsAuth` from `@alfadocs/auth/react` + your server-side BFF — see the `alfadocs-connected-app-oauth` skill). Don't hand-roll the header, sidebar, or connect screen.

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

- **Tokens, not literals** — every colour/spacing/radius/shadow/font uses a `var(--…)` token, never raw hex/rgb/hsl.
- **NEVER invent token names.** The kit's real semantic tokens are `--background`, `--foreground`, `--card`, `--card-foreground`, `--primary`, `--primary-foreground`, `--muted`, `--muted-foreground`, `--border`, `--input`, `--ring`, `--destructive`, `--destructive-foreground`, `--accent`, `--accent-foreground`, `--success`, `--warning`, `--error`, `--info`. Aliases like `--color-background`, `--color-danger`, `--color-text-muted`, `--ds-font-sans` **do not exist** — they silently fall back to hardcoded hex, breaking all four themes. When unsure, check [storybook.alfadocs.com](https://storybook.alfadocs.com) or `src/tokens/index.css`.
- **No inline `style={{}}`** — use Tailwind utility classes referencing `var(--…)` tokens (e.g. `className="bg-[var(--background)] text-[var(--foreground)]"`). Inline styles bypass the theme system entirely.
- **Delete `src/components/ui/`** — Lovable scaffolds a shadcn/ui boilerplate folder by default; delete the whole directory. Import every component from `@alfadocs/ui-kit` instead. Never import from `@/components/ui/`.
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
- Live Storybook: **[storybook.alfadocs.com](https://storybook.alfadocs.com)**
