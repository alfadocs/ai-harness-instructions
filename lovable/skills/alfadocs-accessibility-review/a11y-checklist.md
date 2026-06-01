# AlfaDocs accessibility review — detail & rationale

Expanded notes behind the [SKILL.md](./SKILL.md) checklist. The principle throughout: **`@alfadocs/ui-kit` components are axe-core-clean and ship four themes (including an AAA-accessible one) plus automatic RTL. You inherit all of that for free — the fastest way to lose it is to override the kit or replace its primitives with custom markup.** Live reference: storybook.alfadocs.com.

---

## 1. Why "don't override the kit" is the whole game

The kit is built on Radix primitives, so each component already handles ARIA roles/states, keyboard interaction, focus management, and focus trapping. The common ways an app loses that:

- **Re-implementing a primitive by hand** — a `<div onClick>` "button", a custom dropdown, a bespoke modal. These have no role, no keyboard handling, no focus trap. Use the kit's `Button`, `DropdownMenu`, `Dialog`, etc. instead.
- **Stripping semantics** — removing/overwriting `role`, `aria-*`, or the element type the kit renders; forcing `tabIndex={-1}` on something the user must operate.
- **Suppressing the focus ring** — `outline: none` (or a blanket `*:focus { outline: 0 }`) with no visible replacement. The kit ships a focus-halo token; don't kill it.
- **Breaking the overlay system** — a capture-phase `document` `pointerdown` listener that calls `stopPropagation()` stops Radix from seeing outside-clicks. Result: dialogs/popovers/dropdowns won't dismiss and focus trap misbehaves. This is an accessibility failure, not just a UX one.
- **Missing providers** — `ThemeRoot` (theme + accessible-mode token shifts), `TooltipProvider` (tooltip a11y), `Toaster` (where toasts render). All three belong at the app root.

When auditing, the first question for any oddly-built control is: *does the kit already have this component?* If yes and it wasn't used, that's a blocker.

---

## 2. Keyboard & focus

- **Tab through every screen with no mouse.** Everything operable must be reachable and in a logical order. Hover-only affordances (tooltips that hide an action, menus that only open on hover) fail.
- **Visible focus indicator** on every focusable element. The accessible theme widens the focus ring 2px → 3px — verify in that theme too.
- **Standard keys** per primitive: Esc closes overlays, Enter/Space activate, arrows move within menu/tabs/radio/listbox. If you didn't override the kit you get these automatically.
- **No traps**; focus returns to the triggering element when an overlay closes.
- **Landmarks / skip link** on long pages so AT users can jump past repeated nav.

---

## 3. Target size — and why you must test the accessible theme

The kit's minimum touch/click target is **44px** in light/dark and **48px in the accessible theme** (`<ThemeRoot accessible>` / `theme-accessible`). The accessible theme also raises base type one step. Two failure modes:

1. Custom CSS shrinking a kit control below its size variant.
2. Tight custom layouts that fit at 44px/normal type but **clip or overflow at 48px/larger type**. Always re-walk key screens in light-accessible and dark-accessible.

---

## 4. Colour contrast

The sharp edge specific to this palette:

> **`magenta-500` is the light-mode accent and measures ~3.2:1 on white — it FAILS WCAG AA for normal-size text.** For accent *text* on light surfaces use **`magenta-700` or deeper**. `magenta-500` is fine for large decorative fills and icons. In **dark** mode the accent role is `fuchsia`, which is safe.

Other rules:

- **Prefer semantic aliases** (`--primary`, `--link`, `--success`, `--warning`, `--destructive`, `--error`, `--info`) over raw ramp steps (`--color-violet-500`). Aliases retune per theme; raw steps don't and can fail in one of the four themes.
- **No hard-coded hex/rgb.** Literals bypass the token system, don't shift per theme, and weren't contrast-checked.
- **Check all four themes** — light, dark, light-accessible, dark-accessible. The accessible themes are designed for AAA; the base themes target AA. A pairing that passes light can fail dark.
- **Never rely on colour alone** for status/validation/required — pair with an icon or text.

---

## 5. Accessible names

- **Icon-only buttons** must carry a translated `aria-label` (the visible glyph isn't a name).
- **Inputs** need an associated `<label>` (or `aria-label`/`aria-labelledby`). The kit's field components wire this when you use their label slot — don't bypass it.
- **Images**: meaningful `alt`, or `alt=""` for purely decorative ones.
- **Form errors** associated to the field (`aria-describedby` or the kit error slot) so AT announces them; never colour-only.
- The accessible-name text itself must be **translated** (see §7) — a hardcoded English `aria-label` is both an a11y and an i18n bug.

---

## 6. RTL (Arabic)

The kit mirrors automatically **because it uses CSS logical properties** — your app must too, or RTL breaks at your layer.

- **`dir="rtl"`** on the root/`<html>` when the active locale is RTL (`ar`).
- **Logical properties only:** `margin-inline-start/end`, `padding-inline-start/end`, `text-start/end`, and the `ms-*`/`me-*`/`ps-*`/`pe-*` utilities. **Banned:** `margin-left/right`, `padding-left/right`, `text-left/right`, `ml-*`/`mr-*`/`pl-*`/`pr-*`, and fixed `left:`/`right:` offsets. Physical props don't flip.
- **Spot-check `ar` live:** nothing clipped/overlapping/stuck to the wrong edge; directional icons (chevrons, arrows, back buttons) point the right way.

---

## 7. Reduced motion

- The kit honours `prefers-reduced-motion`, and the **accessible theme sets animation duration to 0ms**. Don't reintroduce unconditional custom transitions / auto-playing animation that ignore the OS preference.
- No essential information should be conveyed **only** through motion (e.g. a transient flash with no persistent state).

---

## 8. Translated strings (a11y + i18n overlap)

- **Every** user-visible string through `useTranslation()` — JSX text, button labels, placeholders, `aria-label`, `alt`, toasts, error/empty/loading states. Hardcoded literals are a blocker (they also can't be read in the user's language by AT).
- **Namespaces:** app strings `app.*`, kit strings `ui.*` — keep them separate; never write app strings into `ui.*`.
- **Production-ready locales** today are `en / it / de` (+ `ar` for RTL); the other supported locales fall back to English for missing keys.
- **CJK:** the kit does not bundle Chinese/Japanese fonts. If the app offers `zh`/`ja`, the Noto Sans SC/JP stylesheet must be added to `<head>`, or text renders as tofu boxes.

---

## Severity quick-reference

| Severity | Examples |
|---|---|
| **Blocker** | hand-rolled control replacing a kit primitive; no visible focus ring; keyboard trap; missing accessible name on a control; `magenta-500` as accent text on white; broken overlay dismissal (capture-phase `stopPropagation`); hardcoded user-visible strings; physical CSS props breaking RTL |
| **Warning** | target under size only in the accessible theme; missing skip link; colour-only status with an available workaround; reduced-motion ignored for non-essential motion; raw ramp step where a semantic alias belongs |
| **Note** | tab order could read cleaner; redundant `aria-label`; minor CJK font gap on a non-priority locale |

**Fix blockers before shipping. Flag warnings and notes with file + component references.** Verdict line: *"Zero blockers — a11y-ready"* or *"N blockers — not ready."*
