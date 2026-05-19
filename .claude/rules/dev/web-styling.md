# Styling

Visual design tokens, color palette, typography, breakpoints, theme tokens, and the responsive system are owned by the **design system** in `rules/design/_core-requirements.md` and its companions (especially `responsive-and-mobile.md`). This file covers only the dev-side CSS implementation choices that the design system does not dictate.

- **SCSS Modules** — required for all component-scoped styles.
- **Container Queries** — prefer over media queries when the layout depends on the component's container width, not the viewport.
- No inline styles except for truly dynamic values (e.g., calculated widths set via JS).
- No CSS-in-JS libraries (styled-components, Emotion, etc.).
- Use `color-mix()` for semi-transparent tints rather than hardcoded `rgba` values.
- Use `min-height: 100dvh` (dynamic viewport height) rather than `100vh` for full-height layouts.
