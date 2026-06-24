# Audit greps & catalogue cross-reference

Fast first-pass sweeps for an AlfaDocs UI design-system review. Run these over the
app source (usually `src/`), then eyeball the hits. None of these are
authoritative on their own — they surface candidates; you confirm each against
[storybook.alfadocs.cloud](https://storybook.alfadocs.cloud) and the rules in
`SKILL.md`. Adjust paths to the repo layout.

## 1. Competing UI libraries (BLOCKER)

Check `package.json` and imports:

```bash
grep -REn "@mui/|@material-ui/|react-bootstrap|[\"']bootstrap[\"']|@chakra-ui/|@mantine/|[\"']antd[\"']" src/ package.json
```

Also flag a `src/components/ui/*` tree of shadcn/Radix copies that duplicate kit
primitives (Button, Dialog, Input, Toast…) — the kit already ships these.

## 2. Raw colour literals (BLOCKER)

```bash
grep -REn "#[0-9a-fA-F]{3,8}\b|rgba?\(|hsla?\(" src/ --include=*.tsx --include=*.ts --include=*.css
```

Every hit in app code is a candidate. The only acceptable colour reference is a
`var(--…)` token (prefer the semantic alias: `var(--primary)`, `var(--accent)`,
`var(--destructive)`, `var(--success)`, `var(--warning)`, `var(--error)`,
`var(--info)`, `var(--link)`).

## 3. Raw spacing / radii / fonts (WARNING)

```bash
grep -REn "font-family|border-radius|(margin|padding)[^-]*:\s*[0-9]" src/ --include=*.css --include=*.tsx
```

Look for `Fraunces`/serif in app screens (marketing-only) and hardcoded `px`/`rem`
spacing or radii where `--spacing-*` / `--radius-*` exist.

## 4. Physical-direction CSS / utilities (WARNING → BLOCKER in bulk)

```bash
grep -REn "margin-(left|right)|padding-(left|right)|text-align:\s*(left|right)|\b(ml|mr|pl|pr)-[0-9]|text-(left|right)\b" src/
```

Replace with logical equivalents: `margin-inline-start/end`,
`padding-inline-start/end`, `text-start/end`, `ms-*`/`me-*`/`ps-*`/`pe-*`.

## 5. Hardcoded user-visible strings (WARNING)

```bash
grep -REn ">[^<>{}\n]*[A-Za-z]{3,}[^<>{}\n]*<" src/ --include=*.tsx
```

Noisy by design — scan for literal labels/headings/placeholders that should be
`t('app.…')`. Also check `placeholder=`, `aria-label=`, `title=` attributes and
toast/alert message strings.

## 6. Namespace pollution (BLOCKER)

```bash
grep -REn "t\(['\"]ui\." src/
```

App code must use `app.*`, never write into the kit's `ui.*` namespace.

## 7. App-root wiring (BLOCKER)

```bash
grep -REn "@alfadocs/ui-kit/tokens|ThemeRoot|TooltipProvider|<Toaster|@alfadocs/ui-kit/i18n" src/
```

Expect: tokens import exactly once; `ThemeRoot` + `I18nextProvider` +
`TooltipProvider` wrapping the app; one `Toaster`; kit i18n registered.

## 8. XSS sink (BLOCKER)

```bash
grep -REn "dangerouslySetInnerHTML" src/
```

Every hit must be inside the kit's `SafeHtml`; anywhere else is a blocker. Raw
HTML → `import { SafeHtml } from '@alfadocs/ui-kit/safe-html'`.

## 9. Overlay-system killer (BLOCKER)

```bash
grep -REn "addEventListener\(['\"](pointerdown|mousedown)['\"].*true|capture:\s*true" src/
```

Inspect any capture-phase `document` `pointerdown`/`mousedown` listener — if it
calls `stopPropagation()` it breaks Dialog/Popover/Dropdown dismissal.

## 10. Heavy components without their peer (BLOCKER — build fails)

If the app imports any of these, confirm the optional peer is in `package.json`:

| Kit component | Required peer |
|---|---|
| Calendar | `@fullcalendar/*` |
| DataTable | `ag-grid-react` |
| Chart / Sparkline | `apexcharts` |
| PaymentForm | `@stripe/*` |
| RichTextEditor | `@tiptap/*` |
| PdfViewer | `pdfjs-dist` |
| SignatureCapture | `signature_pad` |
| PhoneInput | `libphonenumber-js` |

## Catalogue cross-reference

Before calling anything bespoke "necessary", confirm the kit doesn't already cover
it on **storybook.alfadocs.cloud**. The ~125 components span:

| Category | Examples (confirm exact names/props in Storybook) |
|---|---|
| Actions | Button, IconButton |
| Form inputs | TextInput, Select, SearchInput, PhoneInput, Switch, CopyField |
| Data display | Avatar, List, Stat, Timeline, Table |
| Navigation | Tabs, Sidebar, Link, NavigationMenu |
| Feedback & overlay | Alert, Dialog, Popover, Tooltip, NotificationTray |
| Layout | AppFrame, Card |
| AI & chat | assistant / chat primitives |
| Domain-specific | dental-practice components |

If a kit component exists for the use case, the hand-rolled or foreign-library
version is a finding — usually a **blocker**, because it won't inherit the kit's
themes, RTL, or accessibility.
