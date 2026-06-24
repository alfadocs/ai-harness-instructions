---
name: alfadocs-accessibility-review
description: Use when reviewing or auditing an AlfaDocs app for accessibility (a11y) / WCAG — keyboard reachability, visible focus, target size, colour contrast, accessible names, RTL, reduced motion, and translated strings. Triggers on "accessibility review", "a11y audit", "WCAG check", or pre-ship a11y sign-off — NOT for building UI.
---

# AlfaDocs accessibility review

> Accessibility is this skill's own domain — the Engineering Standard doesn't cover a11y. It complements §12 (design system): most a11y wins are inherited from the kit.

A checklist for auditing the accessibility of an AlfaDocs app (Lovable + Supabase, UI built with `@alfadocs/ui-kit`).

**Core principle:** the kit's components are already axe-clean and ship four themes (incl. AAA-accessible) plus RTL support. **Most a11y wins are inherited — most a11y bugs are introduced by overriding the kit.** So the review is mostly: *did we break what was given for free?* Live component reference: **storybook.alfadocs.cloud**.

Walk every screen, dialog, drawer, toast, form, and empty/loading/error state. For deeper rationale on any item, see [a11y-checklist.md](./a11y-checklist.md).

## 1. Don't break the kit's built-in a11y

- [ ] **UI is built from `@alfadocs/ui-kit`** — no Material UI, Bootstrap, shadcn, Chakra, Mantine, or hand-rolled custom controls re-implementing a kit primitive. Hand-rolled controls forfeit the kit's ARIA, keyboard, and focus handling.
- [ ] **Component semantics not overridden** — no stripping the role/ARIA a component renders, no `<div onClick>` standing in for a `Button`, no `tabIndex={-1}` on something the user must reach, no `role` swapped on a kit element.
- [ ] **Overlay system intact** — no capture-phase `document` `pointerdown` listener calling `stopPropagation()`. It silently breaks focus-trap and dismissal in every Dialog / Popover / Dropdown, which is an a11y failure (focus escapes, Esc dead).
- [ ] **Providers wired** — `ThemeRoot` + `TooltipProvider` + `Toaster` present at the root. Missing `TooltipProvider` breaks tooltip a11y; missing `ThemeRoot` drops the accessible-theme token shifts.

## 2. Keyboard reachable + visible focus

- [ ] **Every interactive element reachable by Tab** in a sensible order; nothing operable only by mouse/hover.
- [ ] **Visible focus ring on every focusable element** — never `outline: none` / `*:focus { outline: 0 }` without an equivalent visible replacement. The kit's focus halo token is there; don't suppress it.
- [ ] **Esc / Enter / Space / arrows** behave per the primitive (Esc closes overlays, arrows move within menus/tabs/radios). If you didn't override the kit, this is free.
- [ ] **No keyboard trap** — focus can always leave a dialog/menu; focus returns to the trigger on close.
- [ ] **Skip-to-content** or logical landmark order on long pages.

## 3. Target size

- [ ] **Interactive targets meet the minimum** — the kit sets 44px (light/dark) and **lifts to 48px in the accessible theme**. Don't shrink kit controls below their size variant with custom CSS / cramped wrappers.
- [ ] **Audit in the accessible theme** (`<ThemeRoot accessible>`), where targets are 48px and type is one step larger — custom layouts that "just fit" at 44px often overflow or clip here.

## 4. Colour contrast

- [ ] **Accent text in light mode uses `magenta-700` or deeper.** `magenta-500` (the light-mode accent) is ~3.2:1 on white — **fails WCAG AA** for normal text. Accent is fine for large decorative fills/icons, not for body/label text on light surfaces. (Dark mode the accent is `fuchsia`, which is safe.)
- [ ] **Use semantic aliases, not raw ramp steps** for text — `var(--primary)`, `var(--link)`, `var(--error)` retune per theme; raw `--color-*-500` does not and can fail in one theme.
- [ ] **No hard-coded hex/rgb** that bypasses the tokens — literal colours don't shift per theme and aren't contrast-checked.
- [ ] **Verify in all four themes** (light, dark, light-accessible, dark-accessible). A combo that passes in light can fail in dark.
- [ ] **Info not by colour alone** — status/validation also carries an icon or text label.

## 5. Accessible names

- [ ] **Every control has an accessible name** — icon-only buttons need a translated `aria-label`; inputs need an associated `<label>` (or `aria-label`/`aria-labelledby`); images need meaningful `alt` (or `alt=""` if decorative).
- [ ] **Names are translated** — the `aria-label` text comes from `useTranslation()`, not a hardcoded literal (see §7).
- [ ] **Errors are announced & linked** — form errors associated to their field (`aria-describedby` / kit error slot), not colour-only.

## 6. RTL (Arabic)

- [ ] **`dir="rtl"` flips the layout** when locale is `ar` — set it on `<html>`/root when the active language is RTL.
- [ ] **Logical properties only** — `margin-inline-start/end`, `padding-inline-start/end`, `text-start/end`, `ms-*`/`me-*`/`ps-*`/`pe-*`. **Never** `margin-left`, `text-right`, `ml-*`, `pl-*`, fixed `left:`/`right:`. Physical props don't mirror and break RTL.
- [ ] **Spot-check `ar`** — no overlapping/clipped/left-stuck elements; icons/chevrons that imply direction flip correctly.

## 7. Reduced motion

- [ ] **`prefers-reduced-motion` honoured** — the kit's accessible theme drops animation to 0ms and the kit respects the OS pref. Don't reintroduce unconditional custom transitions/auto-playing motion that ignore the pref.
- [ ] **No essential info conveyed only through motion** (e.g. a flash with no static state).

## 8. Translated strings

- [ ] **All user-visible text goes through `useTranslation()`** — no hardcoded literals in JSX, button labels, placeholders, `aria-label`, `alt`, toast/error messages, empty/loading states.
- [ ] **Namespaces kept separate** — app strings under `app.*`, kit strings under `ui.*`. Don't put app strings in `ui.*`.
- [ ] **CJK note** — if `zh`/`ja` are offered, the Noto Sans SC/JP stylesheet is in `<head>` (kit doesn't bundle CJK); otherwise glyphs render as boxes.

---

## Reporting findings

Report a flat, scannable punchlist. For each finding: **severity · file/area · what's wrong · fix.**

- **Blocker** — fails WCAG or breaks a user who relies on AT: keyboard trap, no focus ring, missing accessible name, `magenta-500` accent text on white, hand-rolled control replacing a kit primitive, broken overlay dismissal, hardcoded user-visible strings, physical props breaking RTL. **Fix blockers before shipping.**
- **Warning** — degraded but usable: target slightly under size in accessible theme, missing skip link, colour-only status with a workaround, reduced-motion not honoured for non-essential motion. **Flag with a recommendation.**
- **Note** — polish / nice-to-have: tab order could be cleaner, redundant `aria-label`, minor CJK font gap. **Flag for later.**

End with a one-line verdict: **"Zero blockers — a11y-ready"** or **"N blockers — not ready, fix listed items."** Cite the file and component for every item so it's directly actionable.
