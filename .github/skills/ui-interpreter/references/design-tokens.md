# Design Token Extraction

How to read colors, typography, spacing, and shape tokens from a UI image.

---

## Colors

### Reading colors from an image

When a precise hex/rgba value is not visible:
1. Describe the color using common CSS color names or nearest Tailwind/Material equivalent.
2. Mark it `[ESTIMATED]`.
3. Note the role it plays (primary, background, border, text, error, success, etc.).

### Token role mapping

| Visual use | Token role |
|------------|------------|
| Most prominent interactive color (button fill, link) | `--color-primary` |
| Danger/error state (red outline, red text) | `--color-error` |
| Success state (green badge, checkmark) | `--color-success` |
| Warning state (amber alert) | `--color-warning` |
| Page background | `--color-surface` |
| Card/panel background (slightly different from page) | `--color-surface-raised` |
| Body text | `--color-text-primary` |
| Secondary/muted text (lighter gray) | `--color-text-secondary` |
| Disabled text | `--color-text-disabled` |
| Border/divider lines | `--color-border` |

### Output format

```
| Role | Value | Source |
|------|-------|--------|
| --color-primary | #1B6EEA [ESTIMATED] | Button fill |
| --color-text-secondary | #6B7280 [ESTIMATED] | Gray label below stats |
| --color-border | #E5E7EB [ESTIMATED] | Table row divider |
```

---

## Typography

### Scale identification

Look for distinct size levels. Most UIs use 4–6 sizes. Name by role.

| Role | Where it appears | Typical size range |
|------|------------------|--------------------|
| `display` | Hero numbers, huge stats | 36px–72px |
| `heading-1` | Page title | 24px–32px |
| `heading-2` | Section title | 18px–24px |
| `heading-3` | Card title, label header | 14px–18px |
| `body` | Paragraph text, list items | 14px–16px |
| `caption` | Helper text, timestamps, footnotes | 11px–13px |

### Font weight mapping

| Visual appearance | CSS value |
|-------------------|-----------|
| Extra thick, almost bold black | `900` (Black) |
| Thick | `700` (Bold) |
| Slightly thick | `600` (SemiBold) |
| Normal | `400` (Regular) |
| Thin | `300` (Light) |

---

## Spacing

### Identifying the base unit

Examine the smallest consistent gap between elements. Common base units:

| Base unit | Indicates |
|-----------|-----------|
| 4px | Dense UI (data grids, dashboards) |
| 8px | Standard web UI (most common) |
| 16px | Comfortable/spacious UI |

Express all spacing as multiples: `base × 1`, `base × 2`, etc.

### Output format

```
Base unit: 8px [ESTIMATED]

| Use | Value |
|-----|-------|
| Card padding | 24px (3×) |
| Button horizontal padding | 16px (2×) |
| Section gap | 32px (4×) |
| Input height | 40px |
```

---

## Shape & Elevation

### Border radius

| Visual appearance | Value |
|-------------------|-------|
| Sharp corners | `0` |
| Very slight rounding | `2px–4px` |
| Standard rounded buttons | `6px–8px` |
| Soft rounded cards | `12px–16px` |
| Fully circular (avatar, FAB) | `50%` or `9999px` |

### Elevation / Shadow

| Visual appearance | CSS pattern |
|-------------------|-------------|
| Flat (no shadow) | None |
| Subtle lift | `box-shadow: 0 1px 3px rgba(0,0,0,0.08)` |
| Card depth | `box-shadow: 0 4px 12px rgba(0,0,0,0.10)` |
| Modal/overlay | `box-shadow: 0 20px 60px rgba(0,0,0,0.20)` |

---

## Existing Design System Check

Before declaring new tokens, search the codebase for:
- CSS variable declarations (`--color-*`, `--spacing-*`)
- Tailwind config (`tailwind.config.js`) or Tailwind classes in components
- Styled-components theme (`ThemeProvider`)
- Token files (`tokens.ts`, `design-tokens.ts`, `colors.ts`)

If tokens exist, reference them by name. Only declare new tokens if the image contains a value not covered by the existing system.
