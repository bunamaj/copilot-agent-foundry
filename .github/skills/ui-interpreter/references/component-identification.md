# Component Identification — Visual to Code Mapping

How to name, type, and specify components from a UI image.

---

## Component Taxonomy

Assign every visible element to one of these canonical types.

### Layout / Structure

| Visual Pattern | Component Type |
|----------------|----------------|
| Outer page wrapper | `Page` / `Screen` |
| Horizontal or vertical band | `Section` / `Container` |
| Two-column side-by-side | `SplitLayout` / `TwoPane` |
| Card-like box with shadow | `Card` |
| Modal/overlay with backdrop | `Modal` / `Dialog` |
| Slide-in panel | `Drawer` / `Sidebar` |
| Collapsible section | `Accordion` |
| Tab row + content area | `Tabs` |
| Popover on hover/click | `Popover` / `Tooltip` |

### Navigation

| Visual Pattern | Component Type |
|----------------|----------------|
| Top bar with logo + links | `Header` / `Navbar` |
| Left or right vertical links | `Sidebar` / `Nav` |
| Horizontal page path | `Breadcrumb` |
| Step indicators | `Stepper` |
| Bottom tab bar (mobile) | `TabBar` |

### Data Display

| Visual Pattern | Component Type |
|----------------|----------------|
| Rows with columns and headers | `Table` / `DataGrid` |
| Repeated cards in a grid | `CardGrid` / `Gallery` |
| Ordered/unordered list | `List` |
| Key-value pairs | `DescriptionList` / `InfoGrid` |
| Number with label below | `Stat` / `MetricCard` |
| Progress bar | `ProgressBar` |
| Chart (line, bar, pie) | `Chart` — note the type |

### Form & Input

| Visual Pattern | Component Type |
|----------------|----------------|
| Single-line text entry | `Input` / `TextField` |
| Multi-line text entry | `Textarea` |
| Dropdown closed/open | `Select` / `Dropdown` |
| Toggle switch | `Switch` |
| Square tick box | `Checkbox` |
| Round single-select | `Radio` |
| Date picker widget | `DatePicker` |
| File upload area | `FileInput` / `DropZone` |
| Range slider | `Slider` |
| Search bar with icon | `SearchInput` |
| Formatted number input | `NumberInput` |

### Feedback & Status

| Visual Pattern | Component Type |
|----------------|----------------|
| Toast / snackbar | `Toast` / `Notification` |
| Inline colored message box | `Alert` / `Banner` |
| Spinning indicator | `Spinner` / `Loader` |
| Gray placeholder shape | `Skeleton` |
| Empty state with illustration | `EmptyState` |
| Overlay blocking interaction | `LoadingOverlay` |

### Actions

| Visual Pattern | Component Type |
|----------------|----------------|
| Solid or outlined rectangle | `Button` — note variant |
| Icon only, no label | `IconButton` |
| Text that looks clickable | `Link` / `TextButton` |
| Row of buttons | `ButtonGroup` |
| Button with dropdown arrow | `SplitButton` |
| Floating action button | `FAB` |

### Typographic

| Visual Pattern | Component Type |
|----------------|----------------|
| Large page title | `Heading` / `H1` |
| Section subtitle | `Subheading` / `H2–H3` |
| Body copy | `Paragraph` |
| Helper / caption text | `Caption` / `HelperText` |
| Inline colored label | `Badge` / `Tag` |

---

## Behavioral Inference Rules

For every interactive component, state the implied behavior explicitly.

### Buttons & Links
- Is there a loading/disabled state visible?
- Is it a submit action or a navigation action?
- Does it trigger a side effect (API call, modal, redirect)?

### Form Fields
- Are there visible validation error states?
- Is there a character count or constraint label?
- Is the field required (asterisk, label clue)?
- Does it have autocomplete or search behavior?

### Lists & Tables
- Is the list sorted? By what column?
- Are rows clickable (selection or navigation)?
- Is there pagination visible (numbers, "Next/Prev")?
- Is there row-level action (edit/delete buttons)?
- Is there a bulk-select checkbox?

### Modals & Drawers
- What triggers it? (Button label is a clue)
- Does it have a close button?
- Does it have a confirm/cancel action pair?
- Is it blocking (backdrop) or non-blocking?

---

## Data Shape Inference

For every data-driven component, describe the minimum data contract needed.

### Rule: go from visual → prop shape

**Example — User Card**
```
Visual: Avatar image, name in bold, role in gray, "Edit" button

Inferred props:
{
  id: string
  avatarUrl: string | null  // null → show initials fallback
  name: string
  role: string
  onEdit: () => void
}
```

**Example — Metric Stat**
```
Visual: "1,234" large, "Orders Today" below in gray, green "+12%" badge

Inferred props:
{
  label: string
  value: number
  trend?: {
    direction: 'up' | 'down' | 'neutral'
    value: string  // "+12%"
  }
}
```

### Rules for data shapes
- Nullable fields: if the empty state is visible (no avatar, no data), mark the field `| null`.
- Arrays: if a list is shown, note the item shape and list constraints.
- Enums: if a status badge has limited colors (green/red/gray), define the enum rather than using `string`.
- Callback props: every button or interactive element needs an `on*` handler.
