# MWS Design System

**Version 1.4** · A navy + pale + accent visual system for enterprise-grade web applications. Saturated colors are targeted spice, not default tools. Every value is a CSS custom property — never hardcoded.

This file is the **constitution**: tokens, critical rules, and the index to the deep specs. Detail for each surface lives in its companion file (see index below). When generating, the pipeline must load DESIGN.md plus the relevant companion(s).

## Mandate: every component is mobile-responsive. No exceptions.

Every component must work at viewports from 320px to 1920px+ in both light and dark theme. Mobile design is **editing, not scaling** — keep, remove, move, reshape, or replace each element. Uniformly shrinking the desktop layout produces a cramped desktop, not a mobile experience. Full principle and 320px gate checklist in `responsive-and-mobile.md`.

## Critical rules (these govern everything)

1. **Theme-stable foreground rule.** Text on pale fills (`--color-pale-*`) and alert fills (`--color-error|success|warning`) is **always navy** — never `var(--text-primary)`. Pale and alert tokens don't shift between themes; `--text-primary` does. Mixing them produces white-on-pale or navy-on-navy depending on theme.
2. **`var(--color-teal)` direct usage is restricted** to sidebar active state and categorical chart series only. Every other interactive accent uses `var(--accent-interactive)` (blue light / teal dark) — primary buttons, AI surfaces, focus rings, selected states, active tabs, single-series chart fills.
3. **Border radius is 2px** (`--radius`) or full pill (`--radius-pill: 999px`). Nothing in between.
4. **Buttons never wrap** (`white-space: nowrap`). Keep labels short.
5. **Hover flips both border AND text** to `var(--accent-interactive)` on clickable text-bearing elements. Border-only is the broken state.
6. **WCAG 2.2 AA is the floor.** Touch targets ≥ 24×24. Focus rings always visible. Color is never the sole indicator of state. Semantic HTML before ARIA.

## Brand voice

Precise. Warm. Confident. Never cute. Editorial in headlines, neutral in UI. Headlines in sentence case; buttons use verb + noun. No exclamation marks in errors. No "Oops!", "Just", "Simply". Error message formula, banned words, capitalization, and full voice rules: `ux-copy-and-microcopy.md` (canonical).

## Color palette

All colors as CSS custom properties on `:root`. Never hardcode hex.

### Primary
| Token | Hex |
|---|---|
| `--color-navy` | `#000042` |
| `--color-blue` | `#0018F2` |
| `--color-white` | `#FFFFFF` |

### Secondary
`--color-magenta` `#F48DFF` · `--color-orange` `#FC561D` · `--color-gold` `#E5AC2E`

### Highlights
`--color-teal` `#00E2C1` (dark-mode accent) · `--color-neon` `#D2FF3E` (max-attention moments only)

### Pale backgrounds — theme-stable, pair with navy text
`--color-pale-blue` `#E2E8FF` · `--color-pale-magenta` `#FFEDFF` · `--color-pale-orange` `#FFE2DE` · `--color-pale-gold` `#F9E9D2` · `--color-pale-success` `#DCF1D2`

### Alerts — background fills only, text always navy
`--color-error` `#FF3333` · `--color-success` `#75D957` · `--color-warning` `#F1E53C`

### Theme tokens (flip per `[data-theme]`)
| Purpose | Light | Dark |
|---|---|---|
| `--bg-page` | `#F7F7FC` | `#13134E` |
| `--bg-surface` | `#FFFFFF` | `#1C1C66` |
| `--bg-sidebar` | navy | `#0D0D46` |
| `--text-primary` | navy | white |
| `--text-secondary` | `rgba(0,0,66,0.65)` | `rgba(255,255,255,0.7)` |
| `--accent-interactive` | blue | teal |
| `--icon-default` | navy | teal |
| `--border-light` | `#E5E5EE` | `#2C2C80` |
| `--border-button` | `#D9D9D9` | `#4D4DA8` |
| `--focus-ring` | blue | teal |
| `--scrim` | `rgba(0,0,66,0.5)` | `rgba(0,0,0,0.6)` |

Internal-only grays (`--color-navy-gray-1|2|3`) **never** appear in client-facing UI.

## Typography

System fonts only. No web fonts. `--font-sans` for body/UI/buttons; `--font-mix` (Georgia) for navigation, display headings, card titles.

| Element | Font | Size | Line-height |
|---|---|---|---|
| Display | Mix | 64pt | 0.95 |
| H1 | Mix | 44pt | 1.05 |
| H2 | Mix | 32pt | 1.1 |
| H3 | Mix | 24pt | 1.2 |
| Body | Sans | 16pt | 1.5 |
| Eyebrow | Sans Light | 13pt | ALL CAPS, 10% tracking |

Display drops to 44pt at ≤768px, 32pt at ≤480px (`word-break: break-word`). Eyebrows and buttons are ALL CAPS; headlines are sentence case; proper nouns are Title Case.

## Spacing, breakpoints, motion

```
--space-1: 4   --space-2: 8   --space-3: 12   --space-4: 16
--space-5: 24  --space-6: 32  --space-7: 48   --space-8: 64   --space-9: 96   (px)

--bp-sm: 640   --bp-md: 768   --bp-lg: 1024   --bp-xl: 1280   --bp-2xl: 1536  (px)

--duration-fast: 100ms   --duration-base: 200ms   --duration-slow: 300ms   --duration-deliberate: 500ms
--ease-standard: cubic-bezier(0.4, 0, 0.2, 1)
--ease-emphasis: cubic-bezier(0.2, 0, 0, 1)
--ease-exit:     cubic-bezier(0.4, 0, 1, 1)
```

Vertical rhythm: `--space-4` between siblings, `--space-6` between sections, `--space-8` between major regions. Sidebar collapses to drawer below `--bp-lg`. Always wrap motion in `@media (prefers-reduced-motion: reduce)` — never `transition: none`.

## Component states (universal)

Every interactive element has six states: **default**, **hover** (per surface), **focus-visible** (2px outline at `--focus-ring`, 2px offset / 3px on buttons), **active** (one step darker, no movement), **disabled** (`opacity: 0.4`, `pointer-events: none`), **loading** (spinner replaces label, dimensions retained). `outline: none` without a visible replacement is forbidden.

## Extrapolating to new components

When the spec doesn't directly cover a component, **compose from existing tokens — never invent**. Defaults: `--bg-surface` background, 1px `--border-light` border, 2px radius, `--space-4`/`--space-5` padding, `--text-primary` body text, `--accent-interactive` for any interactive accent. Run the subtlety test: if it feels visually loud, it's wrong. Full playbook and pre-flight compliance gates: `extrapolating-the-system.md`.

## Companion specs — load alongside DESIGN.md

DESIGN.md is the constitution. The detail for every surface and pattern lives in a companion file. Builds **must** consult these — don't improvise.

| Surface / concern | Companion file |
|---|---|
| Extrapolating to new components (compliance gates) | `extrapolating-the-system.md` |
| Responsive & mobile (320px gate, gotchas, drawer rules) | `responsive-and-mobile.md` |
| App shell & headers (top bar, layout frame) | `app-shell-and-headers.md` |
| Navigation & IA (sidebar, tabs, breadcrumbs) | `navigation-and-ia.md` |
| Forms & input (anatomy, validation, per-input rules) | `forms-and-input.md` |
| Steppers & wizards (multi-step flows) | `steppers-and-wizards.md` |
| Disclosure surfaces (modal, sheet, popover, inline) | `disclosure-surfaces.md` |
| Notifications & feedback (toast, banner, inline alert, badge) | `notifications-and-feedback.md` |
| Loading, empty & error states | `loading-empty-and-error-states.md` |
| Iconography (Phosphor only, sizing, a11y) | `iconography.md` |
| Data visualization & tables (charts, table patterns) | `data-visualization.md` |
| UX copy & microcopy (voice, errors, buttons, empty-state copy) | `ux-copy-and-microcopy.md` |
| Accessibility (WCAG 2.2 AA recipes, ARIA, keyboard) | `accessibility.md` |
| AI — trust & provenance (citations, confidence, AI vs human) | `ai-trust-and-provenance.md` |
| AI — streaming & latency (cursor, auto-scroll, stop) | `ai-streaming-and-perceived-latency.md` |
| AI — tool use & agency (tool calls, approvals, multi-step) | `ai-tool-use-and-agency.md` |
| AI — uncertainty & errors ("I don't know", refusals, hallucination guards) | `ai-uncertainty-and-errors.md` |
| AI — feedback & correction (thumbs, regenerate, edit) | `ai-feedback-and-correction.md` |
| AI — prompting affordances (chips, slash commands, empty state) | `ai-prompting-affordances.md` |

## Anti-patterns — never do this

Hardcoded hex (use tokens). `var(--color-teal)` outside its two allowed uses. Alert colors on text (they're fills; text is navy). `--text-primary` on pale or alert backgrounds. `--color-navy-gray-*` in client-facing UI. Border radius between 2 and 999px. Wrapping buttons. Title Case headlines. "OK"/"Submit"/"Yes"/"No" button labels — use verb + noun. Exclamation marks or "Oops!"/"Just" in errors. Asterisks for required fields (mark optional). Placeholder as the only label. `outline: none` without a replacement. Color as the sole indicator of state. Filled or duotone Phosphor variants. Pie charts >5 slices, 3D charts, dual-axis charts. `body { overflow-x: hidden }` (breaks sticky positioning — use `overflow-x: clip` on `html`). Flex item containing overflow content without `min-width: 0`. Wrapping a table in `overflow-x: auto` without a `min-width` on the table. Auto-executing destructive AI actions. Numeric confidence scores on AI output. Modal feedback prompts. Pale fill for filtered-to-zero empty states (use `--bg-surface` + border). Generic spinner for AI generation (use thinking dots). Generic "Error" message with no recovery action.

---

**Reference implementation:** `artifacts/docs/design/mws-design-system-showcase.html`. Every rule above is demonstrated there in both light and dark mode with working interactions. When in doubt about how a pattern should look or behave, refer to the showcase before generating new code.
