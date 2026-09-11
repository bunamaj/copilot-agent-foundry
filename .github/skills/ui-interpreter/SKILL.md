---
name: ui-interpreter
description: >-
  Analyzes UI screenshots, wireframes, mockups, and design files to identify
  components, layout structure, design tokens, and behavior — then produces
  structured implementation briefs for delegation to specialist agents.
  USE WHEN: given an image of a UI and asked to build it, understand it, or
  delegate its implementation. DO NOT USE FOR: general React/CSS coding without
  an image as input; design system creation without a visual reference;
  non-UI images (charts, diagrams, photos).
---

Systematically analyze a UI image and produce an implementation brief that specialist agents can act on immediately.

## Workflow

Follow these steps in order every time a UI image is provided:

1. **Receive & categorize** — Identify the image type: screenshot, wireframe, mockup, or design file. Note the visual fidelity level. See `references/visual-analysis.md`.
2. **Decompose layout** — Map the top-level layout skeleton (page regions, grid/flex containers, spacing rhythm). See `references/visual-analysis.md`.
3. **Identify components** — Name every distinct UI component visible. Map each to its closest canonical component type. See `references/component-identification.md`.
4. **Extract design tokens** — Pull colors, typography scale, spacing, and border-radius values from what's visible. See `references/design-tokens.md`.
5. **Infer behavior & state** — Identify interactions, conditional visibility, loading/error states, and form validation implied by the UI. See `references/component-identification.md`.
6. **Infer data shapes** — Describe the props and data structures each component needs. See `references/component-identification.md`.
7. **Produce delegation brief** — Write structured implementation briefs for each specialist agent. See `references/delegation-brief.md`.
8. **Delegate** — Hand off to specialist agents using the briefs from step 7.

## Core Instructions

- Always use `read/viewImage` to examine the image before producing any output.
- Never describe the image in prose before analyzing it — go straight to the structured breakdown.
- If the image is low-fidelity (wireframe/sketch), state assumptions explicitly and mark them as `[ASSUMED]`.
- If the image shows multiple screens or states, process each one separately and label them.
- If a design token value cannot be read precisely (e.g., color looks like a blue but no hex is visible), provide a best-guess value and mark it `[ESTIMATED]`.
- Always identify which components already exist in the codebase (search before declaring new ones needed).
- Order the delegation brief so the frontend engineer can work top-down through the component tree.

## Quality Checklist

Before handing off the brief, verify:

- [ ] All visible components are named and typed
- [ ] Layout skeleton is described (not just individual components)
- [ ] At least one behavioral/interaction note per interactive element
- [ ] Data shapes are listed for all data-driven components
- [ ] Assumptions and estimates are labeled
- [ ] Delegation brief sections are complete for each target agent
- [ ] Existing components already in the codebase are identified (not duplicated)

## Output Format

```
## Image Analysis

**Image type:** [Screenshot | Wireframe | Mockup | Design File]
**Screen/View:** [Name or description]

### Layout Skeleton
[Top-level regions and their relationships]

### Component Inventory

| Component | Type | New or Existing | File (if existing) |
|-----------|------|-----------------|--------------------|

### Design Tokens

| Role | Value | Source |
|------|-------|--------|

### Behavior & State Notes
[Per-component interaction notes]

### Data Shapes
[Per-component prop/data requirements]

---

## Delegation Briefs

### → Frontend Engineer
[Full implementation spec]

### → Test Writer
[Test scenarios and edge cases]

### → Backend Engineer (if needed)
[API contract or data requirements]
```
