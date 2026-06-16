---
name: alfadocs-design-review
description: Use when reviewing or auditing the UI / visual design / design-system compliance of an AlfaDocs app — verifying every screen is built from @alfadocs/ui-kit (no Material UI, Bootstrap, shadcn, Chakra, Mantine, or hand-rolled CSS), uses var(--…) tokens, logical properties, useTranslation, ThemeRoot/four themes, and SafeHtml. Do NOT use this for building UI (see alfadocs-design-system).
---

# AlfaDocs design-system compliance review

> **Enforces Engineering Standard** (workspace knowledge) §12. Cite the section number when you flag a violation.

A pass/fail audit of an AlfaDocs app's UI against `@alfadocs/ui-kit`. This is the **review counterpart** to the `alfadocs-design-system` build skill — it does not contradict it, it enforces it. The source of truth for what exists is the live catalogue: **[storybook.alfadocs.com](https://storybook.alfadocs.com)**.

Work through the checklist below in order. The fastest signal comes from a few `grep`-style sweeps over the source (see [audit-greps.md](./audit-greps.md)), then a manual look at the app root and any flagged components.

## 0. Scope the review first

- Look at `package.json` dependencies and the app's component/page source (usually `src/`).
- Note which screens exist so you can confirm each one renders with the kit, not just the entry point.

## 1. Every piece of UI comes from `@alfadocs/ui-kit` — BLOCKER

The single most important rule. If the kit has a component for it, the app must use it.

- [ ] **No competing UI library** in `package.json` or imports: `@mui/*` / `@material-ui/*`, `bootstrap` / `react-bootstrap`, `@chakra-ui/*`, `@mantine/*`, `antd`, and bare `shadcn`/Radix-copied `components/ui/*` that duplicate kit primitives. Any of these → **blocker**.
- [ ] **No hand-rolled replacements** for things the kit already ships (custom `<button className="btn">`, bespoke modal, hand-built table, custom toast). Cross-check the category list in [audit-greps.md](./audit-greps.md) against storybook.alfadocs.com — if a kit component exists for it, a hand-rolled one is a **blocker**.
- [ ] **App chrome uses the shell, not hand-rolled** — the header, sidebar, account menu, and login screen come from `MarketplaceAppShell` + `ConnectWithAlfadocs` (`@alfadocs/ui-kit/patterns/marketplace-app-shell`, Storybook `Patterns/Public/MarketplaceAppShell`), not a custom or shadcn/Material/Bootstrap layout. A hand-rolled app frame, top-bar, sidebar, or login screen → **blocker**. The shell's "{productName} by Alfadocs" brand lockup (kit 0.52.0+) overridden or swapped for a custom logo → **warning**.
- [ ] **Per-component subpath imports** preferred (`@alfadocs/ui-kit/button`), not deep internal paths (`@alfadocs/ui-kit/dist/...`) or barrel-only when optional peers are missing. Deep `/dist/` import → **warning**. Heavy component (Calendar, DataTable, Chart, PaymentForm, RichTextEditor, PdfViewer) used without its optional peer installed → **blocker** (it will fail to build).

## 2. Tokens, not literals — BLOCKER on colour, WARNING on the rest

- [ ] **No raw colours** — no `#hex`, `rgb()`, `rgba()`, `hsl()`, or named CSS colours in component/style code. Every colour is a `var(--…)` token (semantic alias preferred: `var(--primary)`, `var(--accent)`, `var(--destructive)`, etc.). Raw colour → **blocker**.
- [ ] **No raw spacing / radii / shadows / fonts** — these reference `--spacing-*`, `--radius-*`, shadow tokens, `--font-sans`. Hardcoded `px`/`rem` spacing or radii where a token exists → **warning**.
- [ ] **Accent contrast** — the magenta accent (`magenta-500` / `var(--accent)`) fails AA on white, so accent used as **text** in light mode must be `magenta-700`+; flag accent-coloured body text on light surfaces → **warning**.
- [ ] **`--font-serif` (Fraunces) never in app UI** — it is marketing-only. Serif in an in-app screen → **warning**.

## 3. CSS logical properties only — WARNING

RTL (`ar`) correctness depends on this; the kit gets it for free, the app must not break it.

- [ ] No physical-direction CSS or utilities: `margin-left`/`margin-right`, `padding-left`/`padding-right`, `text-align: left`/`right`, `left:`/`right:` for flow positioning, and Tailwind `ml-*`/`mr-*`/`pl-*`/`pr-*`/`text-left`/`text-right`. Use `margin-inline-start/end`, `padding-inline-start/end`, `text-start/end`, `ms-*`/`me-*`/`ps-*`/`pe-*`. Each occurrence → **warning** (cluster of them across the app → **blocker**, RTL is broken).

## 4. All user-visible strings via `useTranslation()` — WARNING

- [ ] No hardcoded user-visible text in JSX (button labels, headings, placeholders, toast messages, aria-labels). Each runs through `t('app.…')`. Hardcoded strings → **warning** (a screen with no i18n at all → **blocker** for that screen).
- [ ] **Namespace hygiene** — app strings live under `app.*`; the app must **not** write into the kit's `ui.*` namespace. App keys polluting `ui.*` → **blocker**.
- [ ] Kit i18n is registered (`import '@alfadocs/ui-kit/i18n'` or equivalent bundle wiring) so kit components aren't showing raw keys. Missing → **warning**.

## 5. ThemeRoot wired + all four themes work — BLOCKER on wiring

- [ ] **Tokens imported exactly once** at the app root: `import '@alfadocs/ui-kit/tokens';`. Missing → **blocker** (everything renders unstyled).
- [ ] **App wrapped** in `ThemeRoot` + `I18nextProvider` + `TooltipProvider`, with `Toaster` mounted once inside the tree. Missing `ThemeRoot` → **blocker**; missing `TooltipProvider` while tooltip-bearing components are used → **blocker**; missing `Toaster` while toasts are fired → **warning**.
- [ ] **All four themes render** — light, dark, light-accessible, dark-accessible. Manually exercise the theme switch (or `useTheme()`). Watch for: app colours that don't shift in dark (a tell-tale of hardcoded literals), unreadable contrast in dark/accessible, layout breaking in accessible (target sizes grow 44→48px, type up one step). Broken dark or accessible theme → **blocker**.
- [ ] **No theme overrides fighting the system** — app CSS must not pin `color-scheme`, force a background colour outside tokens, or hardcode a theme class. → **warning**.

## 6. SafeHtml for any raw HTML — BLOCKER

- [ ] No `dangerouslySetInnerHTML` anywhere outside `SafeHtml`. Any direct use → **blocker** (XSS sink). Raw/user HTML must go through `import { SafeHtml } from '@alfadocs/ui-kit/safe-html'`.

## 7. Don't fight the overlay system — BLOCKER if present

- [ ] No capture-phase `document` `pointerdown`/`mousedown` listener that calls `stopPropagation()`. It starves the outside-click detector behind every dismissable primitive (Dialog, Popover, DropdownMenu) and leaves them stuck open. → **blocker**.

## 8. Cross-check against the live catalogue

- [ ] For anything that looks bespoke, confirm on **storybook.alfadocs.com** whether a kit component, variant, or prop already covers it. If it does, the bespoke version is at least a **warning** (re-implementing the kit) and usually a **blocker** (it won't theme/RTL/a11y-match).
- [ ] Confirm component **variants and states** used by the app actually exist in the kit (don't invent a `variant` the kit doesn't expose).

---

## How to report findings

Produce one merged punchlist, grouped by severity, each item citing the **file/area** (and line where you can):

- **Blocker** — must fix before shipping. Competing UI library, hand-rolled replacement of a kit component, raw colour literal, missing `tokens` import, missing `ThemeRoot`, broken dark/accessible theme, `dangerouslySetInnerHTML` outside `SafeHtml`, the capture-phase overlay-killer, app strings in `ui.*`, a heavy component used without its peer.
- **Warning** — should fix. Hardcoded spacing/radii, physical-direction CSS, hardcoded strings, serif in app UI, accent-as-text on light, deep `/dist/` import, missing `Toaster`/kit i18n registration.
- **Note** — informational (a kit component would simplify this, a semantic alias would be cleaner than a raw ramp step).

Then: **fix the blockers** in place (swap the competing component for the kit one, replace literals with tokens, wire `ThemeRoot`/tokens, route HTML through `SafeHtml`) and **flag the warnings/notes** for the author with the citation. If zero blockers, say so explicitly and confirm the UI is design-system compliant. Re-run the relevant greps after fixing to confirm the count is zero.
