# Pick the right @alfadocs/ui-kit component

The most common mistake is reaching for the obvious primitive (`TextInput`,
`Button`, `Dialog`) when a **dedicated** component already handles the case — with
accessibility, i18n, validation, and theme-shifts built in. Before you build a UI
element, find its row below and use that component. If nothing fits, see
["When nothing fits"](#when-nothing-fits) — do **not** hand-roll.

The live, authoritative catalogue (every variant + prop) is
**[storybook.alfadocs.cloud](https://storybook.alfadocs.cloud)**.

## Form inputs

| You're building | Use | Not |
|---|---|---|
| Plain text field | `TextInput` | — (baseline) |
| Email field | `EmailInput` | `TextInput type=email` |
| Password field | `PasswordInput` (reveal, caps-lock, strength) | `TextInput type=password` |
| One-time / verification code | `OtpInput` | N × `TextInput` |
| Phone with country code | `PhoneInput` | `Select` + `TextInput` |
| Number field | `NumberInput` | `TextInput type=number` |
| Date / time / datetime | `DatePicker` / `TimePicker` / `DateTimePicker` | native `<input type=date>` |
| Date range | `DateRangePicker` | two date inputs |
| Pick from a fixed list | `Select` | styled `<input>` |
| Pick several from a list | `MultiSelect` | — |
| Type-to-pick from a known list | `Combobox` | — |
| Freeform text with hint suggestions | `Autocomplete` | — |
| Search box | `SearchInput`; with results popover `SearchBar`; full ⌘K `CommandPalette` | — |
| Multi-line text | `TextArea` | `<textarea>` |
| File / drag-drop upload | `FileUpload` | `<input type=file>` |
| Checkbox / group | `Checkbox` / `CheckboxGroup` | — |
| Radios | `RadioGroup` (or `Radio` solo) | — |
| On/off toggle (applies immediately) | `Switch` | `Checkbox` |
| Slider / colour / signature / Stripe | `Slider` / `ColorPicker` / `SignatureCapture` / `PaymentForm` | — |
| **Any field with a label + help + error** | wrap it in `FormField` | hand-rolled `<label>`+`<input>`+`<span>` |

## Actions

| You're building | Use |
|---|---|
| Text button | `Button` (`primary`/`secondary`/`outline`/`ghost`/`destructive`) |
| Icon-only button | `IconButton` (needs `aria-label`) |
| Segmented / grouped buttons | `ButtonGroup` / `IconButtonGroup` |
| Link styled as a button | `Button asChild` + `<Link>`, or `Link variant="button"` |
| Body-text hyperlink | `Link` |
| Mobile floating action | `FloatingActionButton` |
| **The full pre-auth connect screen** (card + lockup + button) | `ConnectWithAlfadocs` from `@alfadocs/ui-kit/patterns/marketplace-app-shell` — props: `productName`, `title`, `description`, `connectLabel`, `status` (`idle`/`connecting`/`error`), `onConnect`. Wire `onConnect` to `useAlfadocsAuth().connect()`. **`productName` is rendered by `ProductLockup` inside a 400px card** — keep it short (≤20 chars) or it clips. |
| **Standalone branded "Sign in" button only** | `SignInWithAlfadocsButton` — renders the AlfaDocs brand mark + label (like "Sign in with Google"). Use inside your own screen layout when you don't want the full `ConnectWithAlfadocs` card. |

## Data display

| You're building | Use |
|---|---|
| Status pill | `Badge` · removable chip `Tag` |
| Person photo/initials | `Avatar` |
| Single metric | `Stat` · key/value `KeyValuePair` · list `DescriptionList` |
| Table (sort/filter/page, or >1000 rows) | `DataTable` (virtualised) — never hand-render rows |
| Rating | `Rating` · trend chart `Sparkline` · full chart `Chart` · events `Timeline` · list `List` |
| Loading | `Skeleton` (layout-stable) — not a "Loading…" string |
| Empty / no-data / error state | `EmptyState` |
| **Practice contact block (address + VAT + phone)** | `ContactCard` (owns locale `vat`/`tel:` wiring) |
| Financial state pill | `TransactionChip` · consent gating `PrivacyLock` · paywall `FreemiumPaywall` |
| SR-only text | `VisuallyHidden` — never `display:none` |

## Feedback, status, overlays

| You're building | Use |
|---|---|
| Persistent page banner | `Alert` |
| Transient toast ("Saved") | `Toast` |
| Modal sub-task (Save/Cancel) | `Dialog` |
| **Destructive confirm ("Are you sure?")** | `AlertDialog` — never `Dialog` |
| Side panel | `Sheet` |
| Hover hint (non-interactive) | `Tooltip` |
| Click content **with buttons/inputs** | `Popover` — not `Tooltip` |
| Actions menu / right-click | `DropdownMenu` / `ContextMenu` |
| Spinner / progress | `Spinner` / `Progress` |

## Navigation & layout

`Breadcrumb` · `Pagination` · `Sidebar` · `NavigationMenu` · `SkipLink` ·
`StepperAccordion` / `StepperProgress` · **appointment funnel → `Booking`** ·
`Tabs` · `Collapsible`/`Accordion` · `Resizable` · `ScrollArea` · `Separator` (not `<hr>`) ·
app shell → `AppFrame` · `AspectRatio`.

## AI / chat

`ChatInput` · `AiPromptInput` · `ChatMessage` · `ChatContainer` · `TypingIndicator` ·
`SuggestionChip` · `AudioVisualiser` · `AudioRecorder` · `TranscriptPanel`.

## Marketplace app shell (kit 0.36+)

Building a whole app on AlfaDocs? Don't assemble the frame yourself — the kit ships it.
Import from `@alfadocs/ui-kit/patterns/marketplace-app-shell`.

| You're building | Use | Key props |
|---|---|---|
| The whole authed app frame | `MarketplaceAppShell` | `productName`, `nav`, `user`, `onSignOut` |
| The full pre-auth screen (card + lockup + CTA) | `ConnectWithAlfadocs` | `productName` (≤20 chars — clips in the 400px card), `title`, `description`, `connectLabel`, `status`, `onConnect` |
| A standalone branded OAuth CTA button | `SignInWithAlfadocsButton` | `onPress`, `loading`, `label` |

`ConnectWithAlfadocs` is the **whole screen** (renders `min-h-dvh`). `SignInWithAlfadocsButton` is just the **button** — use it when you're building your own layout. Both are auth-decoupled; wire to `useAlfadocsAuth()` from `@alfadocs/auth/react`.

## Anti-patterns — stop and switch

- ❌ `<input type=password>` + custom show/hide → ✅ `PasswordInput`
- ❌ 6 × `<input maxLength={1}>` → ✅ `OtpInput`
- ❌ `<select>` + `<input type=tel>` → ✅ `PhoneInput`
- ❌ two date inputs for a range → ✅ `DateRangePicker`
- ❌ `<Tooltip>` containing a button → ✅ `Popover`
- ❌ `<Dialog>` for "are you sure?" → ✅ `AlertDialog`
- ❌ `<div class="alert">` → ✅ `Alert`
- ❌ hand-rolled VAT/phone block → ✅ `ContactCard`
- ❌ spinner + "Loading…" → ✅ `Skeleton`
- ❌ fake `<span class="link">` → ✅ `Link`
- ❌ `display:none` for SR text → ✅ `VisuallyHidden`
- ❌ 10k `<tr>` rendered by hand → ✅ `DataTable`
- ❌ a `<Button>` faked into the AlfaDocs sign-in lockup → ✅ `SignInWithAlfadocsButton`
- ❌ hand-rolled app header + sidebar + account menu → ✅ `MarketplaceAppShell` (kit 0.36+)
- ❌ a custom "Connect with AlfaDocs" landing screen → ✅ `ConnectWithAlfadocs` (full screen pattern)
- ❌ a plain `Button` labelled "Connect with AlfaDocs" → ✅ `SignInWithAlfadocsButton` (branded lockup)
- ❌ `productName="Today's Wallboard"` in `ConnectWithAlfadocs` without checking width → keep `productName` ≤20 chars or it clips inside the 400px card

## When nothing fits

1. Re-scan this list — most "no fit" calls are the wrong row, not a missing component.
2. If you must ship, use the **closest** kit primitive (never a bare `<div>` / hand-rolled
   markup) and leave a `// TODO(ds): need <X>` comment. Flag the gap to the AlfaDocs team.
