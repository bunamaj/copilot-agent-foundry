# Visual Analysis — Reading UI Images

How to systematically decompose a UI image into a structured, agent-readable breakdown.

## Image Type Classification

Classify the image before doing anything else. Fidelity determines how many assumptions are required.

| Type | Fidelity | Signals | Assumption Overhead |
|------|----------|---------|---------------------|
| Screenshot | High | Real fonts, real colors, real data | Minimal |
| Design file export (Figma, Sketch) | High | Pixel-perfect, named layers may be visible | Minimal |
| High-fidelity mockup | High | Styled but possibly static/fake data | Low |
| Low-fidelity mockup | Medium | Gray boxes, placeholder text, rough typography | Medium |
| Wireframe | Low | Outlines, no color, no real content | High |
| Hand-drawn sketch | Very low | Loose shapes, arrows, labels | Very high |

**Rule:** If fidelity is Medium or lower, prefix every token value with `[ESTIMATED]` and every structural guess with `[ASSUMED]`.

---

## Layout Skeleton Decomposition

Always start with the macro structure before zooming into components.

### Step 1 — Identify top-level regions

Common page regions:

```
┌─────────────────────────────┐
│         Header / Nav         │  ← sticky, fixed, or inline
├───────────┬─────────────────┤
│  Sidebar  │   Main Content  │  ← sidebar may not exist
│           │                 │
│           ├─────────────────┤
│           │   Footer        │
└───────────┴─────────────────┘
```

Region names to use: `header`, `sidebar`, `main`, `footer`, `modal-overlay`, `drawer`, `panel`, `toolbar`.

### Step 2 — Identify the layout model per region

| Visual Signal | Layout Model |
|---------------|--------------|
| Items side-by-side, equal or weighted widths | Flexbox row |
| Items stacked vertically | Flexbox column |
| Items in a grid (rows × columns) | CSS Grid |
| Cards flowing/wrapping | Flex wrap or Grid auto-fill |
| Absolute positioned elements | Position absolute/fixed |

### Step 3 — Measure spacing rhythm

Scan the gaps between elements and try to identify a base unit (4px, 8px, or 16px systems are most common). Express all spacing as multiples: `base × 1`, `base × 2`, etc.

### Step 4 — Identify scroll behavior

- Does the header stay fixed while content scrolls?
- Is there a scrollable list or table within the main content?
- Are there sticky column headers?

---

## Describing Regions to Agents

```
### [Region Name]
- **Layout:** flex-row | flex-column | grid | absolute
- **Dimensions:** full-width | [estimated px or %] | responsive breakpoint noted
- **Spacing:** [padding and gap values]
- **Contents:** [comma-separated component names]
- **Scroll:** none | vertical | horizontal
```

---

## Multiple Screens or States

If the image shows more than one screen or state:

1. Label each panel: `[Screen A]`, `[Screen B]`, or use the UI's own labels.
2. Process each panel independently.
3. After both are processed, note the differences and describe the trigger that moves between them.

---

## Common Misreadings to Avoid

| What you might see | What it actually is |
|--------------------|---------------------|
| A box with a drop shadow | A Card component |
| A colored rectangle at the top | A Header or Banner |
| A thin horizontal line | A Divider or border-bottom |
| Circular user image | Avatar component |
| Colored dot or badge | Badge or Status Indicator |
| Grayed-out button | Disabled state, not a separate component |
| Repeated rows of similar content | A List or Table with mapped data |
| Placeholder gray box | Image or skeleton loader |
