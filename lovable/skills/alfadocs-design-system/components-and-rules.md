# Component catalogue & full consumer rules

Reference detail for `@alfadocs/ui-kit`. The **live, current** catalogue — every variant, state, and prop — is the Storybook at **[storybook.alfadocs.cloud](https://storybook.alfadocs.cloud)**. Browse it before building so you reuse what exists instead of reinventing it.

## Component categories (~125 components)

| Category | Examples |
|---|---|
| **Actions** | Button, IconButton |
| **Form inputs** | TextInput, Select, SearchInput, PhoneInput, Switch, CopyField |
| **Data display** | Avatar, List, Stat, Timeline, Table |
| **Navigation** | Tabs, Sidebar, Link, NavigationMenu |
| **Feedback & overlay** | Alert, Dialog, Popover, Tooltip, NotificationTray |
| **Layout** | AppFrame, Card |
| **AI & chat** | assistant / chat primitives |
| **Domain-specific** | dental-practice components |

This is an overview, not the whole list — confirm exact names and props in the Storybook.

## Importing

Prefer **per-component subpath imports** — they tree-shake without pulling in heavy optional peers:

```tsx
import { Button } from '@alfadocs/ui-kit/button';
import { Dialog } from '@alfadocs/ui-kit/dialog';
```

The root barrel works but only tree-shakes safely when every optional peer is installed, so subpaths are the safer default.

## Components that need an extra optional peer

Install the listed package alongside the kit when you use these:

| Component | Also install |
|---|---|
| Calendar | `@fullcalendar/*` |
| DataTable | `ag-grid-react` |
| Chart / Sparkline | `apexcharts` |
| PaymentForm | `@stripe/*` |
| RichTextEditor | `@tiptap/*` |
| PdfViewer | `pdfjs-dist` |
| SignatureCapture | `signature_pad` |
| PhoneInput | `libphonenumber-js` |

## Rendering raw HTML

`SafeHtml` is the **only** sanctioned way to render raw HTML — it sanitises first:

```tsx
import { SafeHtml } from '@alfadocs/ui-kit/safe-html';

<SafeHtml html={userProvidedHtml} />;
```

Never use `dangerouslySetInnerHTML` directly.

## Tokens in detail

Everything visual is driven by CSS custom properties from `@alfadocs/ui-kit/tokens`. Reference them with `var(--…)`.

### Colour — closed palette (10 ramps × 10 steps)

- **Brand:** `blue`, `violet`, `purple`, `magenta`, `fuchsia`, `grey`
- **Semantic:** `green`, `red`, `yellow`, `indigo`, `orange`

Use a ramp step directly (`var(--color-violet-500)`) or a **semantic alias** that retunes per theme — prefer the alias:

```
var(--primary)   var(--accent)    var(--link)
var(--success)   var(--warning)   var(--destructive)
var(--error)     var(--info)
```

Note: `magenta-500` fails AA on white, so for **text** in light mode the accent uses `magenta-700` or deeper.

### Spacing, radii, shadows, gradients

- **Spacing:** `--spacing-none … --spacing-2xl` (base unit 4px).
- **Radii:** `--radius-sm` (4px), `-md` (8px), `-lg` (16px), `-full`.
- **Shadows:** seven elevation steps, plus chrome variants and a focus halo.
- **Gradients:** page washes, hero bands, surface tints (RTL-aware via direction tokens).

### Typography

Ten type roles (`title-hero`, `title-page`, `title-section`, `title-card`, `body`, `label`, `eyebrow`, `meta`, `metric`…). In-app text uses **Space Grotesk** (`--font-sans`) with Noto fallbacks for non-Latin scripts. `--font-serif` (Fraunces) is **marketing-only** — never in app UI.

## Theme shifts

- **Dark** — the primary colour inverts (violet → fuchsia); magenta stays in the accent role.
- **Accessible** — focus ring 2 → 3px, minimum target 44 → 48px, body/label type up one step, animations → 0ms.

## Coexisting with legacy CSS

If you load the kit alongside legacy styles, declare cascade layers up-front so tokens and components resolve predictably:

```css
@layer legacy, ui-kit-tokens, ui-kit-components;
@import url('./legacy.css') layer(legacy);
@import '@alfadocs/ui-kit/tokens';
```

## Full consumer rules

These mirror the design system's own hard constraints. Break them and you'll get contrast failures, broken dark/accessible modes, or layouts that don't mirror in RTL.

- **Tokens, not literals.** Every colour, spacing, radius, shadow, and font references a `var(--…)` token — never raw hex/rgb/hsl. The palette is closed.
- **Logical properties only.** Use `margin-inline-start/end`, `padding-inline-start/end`, `text-start/end` (and Tailwind `ms-*` / `me-*` / `ps-*` / `pe-*`) — never `margin-left`, `text-right`, `ml-*`, `mr-*`, `pl-*`, `pr-*`. This is what makes RTL work.
- **All user-visible strings through `useTranslation()`.** Kit strings live under `ui.*`; your app's strings under `app.*`. Keep them separate.
- **In-app font is `--font-sans`** (Space Grotesk). `--font-serif` (Fraunces) is marketing-only.
- **Raw HTML only via `SafeHtml`** — never `dangerouslySetInnerHTML` directly.
- **Don't fight the overlay system.** Never add a capture-phase `document` `pointerdown` listener that calls `stopPropagation()` — it starves the outside-click detector behind every dismissable primitive (dialogs, popovers, dropdowns) and leaves them stuck open.
- **No competing UI libraries.** No Material UI, Bootstrap, shadcn/ui, Chakra, or hand-rolled CSS. Use the kit's components and tokens.
- **Kit-first — never invent.** If the kit ships a component for it (it has ~175), use that component; hand-rolling or approximating one it already has — a `<div>` badge, a hand-built table, a custom modal, an avatar circle — is a defect. Look it up in the catalogue first; if something is genuinely missing, STOP and ask, don't fake it with styled `<div>`s or pull in another library.
- **People go through `Avatar`.** Every person/name (patient, operator, staff, user, list row, author) renders the kit `Avatar` for the visual identity — never a hand-rolled circle, hand-written initials, or a raw `<img>`. Pass the person's `name` (and `src` for a photo); `Avatar` derives consistent initials + a deterministic brand colour. Use ONE name format everywhere — the same `fullName` string for `Avatar`'s `name` and the visible label. For a fuller person block use `ContactCard` / `contact-profile-card`.

Follow these and your app inherits the kit's accessibility (axe-core-clean components), all four themes, and right-to-left support for free.
