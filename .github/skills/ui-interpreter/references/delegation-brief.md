# Delegation Brief Format

How to write structured implementation briefs that specialist agents can act on immediately.

---

## Purpose

A delegation brief is a complete, self-contained specification that a receiving agent can execute without needing to re-analyze the image or ask clarifying questions.

---

## Frontend Engineer Brief

```markdown
### → Frontend Engineer

**Context:**
[1–2 sentences: what screen this is, where it lives in the app, what user action leads to it]

**Existing components to reuse:**
[List any components already in the codebase that can be used as-is or with minor props changes]
[If none found: "No matching existing components identified"]

**New components to create:**
| Component | Type | Suggested file path |
|-----------|------|---------------------|
| ... | ... | ... |

**Component tree:**
PageWrapper
  └─ Header
  └─ MainSection
       ├─ FilterBar
       └─ DataTable

**Per-component specs:**

#### ComponentName
- **File:** `src/components/ComponentName.tsx`
- **Props:**
  ```ts
  interface ComponentNameProps {
    // ...
  }
  ```
- **Behavior:** [Interactions, state, event handlers]
- **Styles:** [Layout model, key dimensions, token references]
- **Notes:** [Edge cases, assumptions, constraints]

**Design tokens:**
[Paste the token table from the image analysis]

**Acceptance criteria:**
- [ ] [Specific, testable criteria]
```

---

## Test Writer Brief

```markdown
### → Test Writer

**Context:**
[Same screen context as the frontend brief]

**Components to test:**
[List component names with their file paths]

**Test scenarios per component:**

#### ComponentName

| Scenario | Input / State | Expected Output |
|----------|--------------|-----------------|
| Renders with required props | Valid props | Component renders without error |
| Shows empty state | `items=[]` | Empty state message is visible |
| Handles click | User clicks button | `onAction` callback called |
| Shows disabled state | `disabled=true` | Button has disabled attribute |
| Shows loading state | `isLoading=true` | Spinner visible |
| Shows error state | `error="message"` | Error message rendered |

**Edge cases to cover:**
- Null/undefined prop handling
- Empty string vs null
- Long text / overflow
- Rapid-click / double-submit prevention

**Interaction tests:**
[User flows that span multiple components]

**Accessibility checks:**
- Keyboard navigation through interactive elements
- ARIA labels on icon-only buttons
- Focus management in modals
```

---

## Backend Engineer Brief

Only include if the UI reveals an API requirement not currently met.

### When to include

Include if:
- A new data shape is needed that doesn't match any existing API response
- A new mutation (create, update, delete) has no existing endpoint
- Pagination, filtering, or sorting requires query params not currently supported

```markdown
### → Backend Engineer

**Context:**
[What the UI needs and why current APIs are insufficient]

**New endpoints needed:**

#### GET /api/v1/[resource]
- **Purpose:** [What the UI uses this for]
- **Query params:** [List with types]
- **Response shape:**
  ```ts
  {
    data: Array<{ id: string; /* ... */ }>
    total: number
    page: number
  }
  ```

**Acceptance criteria:**
- [ ] [Specific, testable criteria]
```

---

## Delegation Completeness Checklist

Before finalizing briefs:

- [ ] Frontend brief covers every new component with props, behavior, and style specs
- [ ] Every interactive element has a behavioral spec
- [ ] Test brief has at least one scenario per component
- [ ] Test brief covers empty, loading, and error states for data-driven components
- [ ] If new API data is needed, backend brief is included
- [ ] Acceptance criteria are specific enough to verify

---

## Handoff Sequencing

Delegate in this order:

1. **Frontend Engineer first** — implement components
2. **Test Writer second** — write tests (or in parallel for TDD)
3. **Backend Engineer only if needed** — can run in parallel if API contract is clear

For TDD workflows, invert 1 and 2: Test Writer writes failing tests, Frontend Engineer implements to make them pass.
