---
name: react19-pro
description: >-
  Comprehensively reviews React 19 code for best practices on new APIs (use, useActionState,
  useFormStatus, useOptimistic, cache), Actions, ref-as-prop, Context-as-provider, React Compiler,
  async transitions, form handling, optimistic UI, TypeScript typing changes, and testing with
  React Testing Library. Use when reading, writing, or reviewing React 19.x frontend code.
  Trigger keywords: React 19, useActionState, useOptimistic, useFormStatus, use(), Actions,
  React Compiler, babel-plugin-react-compiler, ref as prop, forwardRef migration, react-dom/client.
  DO NOT USE FOR: React Native, React 18 or earlier, or non-React frameworks.
---

Review React 19.x code for correctness, idiomatic patterns, and breaking change compliance. Version target: **React 19.2.x**.

## Review Process

1. **Load reference files** — consult the table below to identify which reference files apply to the code being reviewed; read only those files.
2. **Check for removed APIs** — scan for `ReactDOM.render`, `forwardRef`, `<Context.Provider>`, `defaultProps`, string refs, `react-dom/test-utils` imports. See `references/migration.md`.
3. **Check new API usage** — verify `use()`, `useActionState`, `useFormStatus`, `useOptimistic` follow their documented constraints. See `references/api.md`.
4. **Check patterns** — verify form handling, optimistic UI, and async transitions follow idiomatic patterns. See `references/patterns.md`.
5. **Check performance** — flag manual `useMemo`/`useCallback`/`memo` if React Compiler is enabled; ensure Rules of React are followed. See `references/performance.md`.
6. **Check TypeScript** — verify ref typing, `useRef` argument, `ReactElement.props`, scoped JSX namespace. See `references/typescript.md`.
7. **Check tests** — verify `act` import source, async test patterns, and RTL usage. See `references/testing.md`.

## Reference Files

| File | Contents | When to load |
|------|----------|-------------|
| `references/api.md` | `use()`, `useActionState`, `useFormStatus`, `useOptimistic`, `cache()`, ref-as-prop, Context-as-provider, ref cleanup, Server Actions | When reviewing new React 19 API usage |
| `references/patterns.md` | Async transitions, form Actions, optimistic UI, Suspense + `use()`, combined `useActionState` + `useOptimistic` | When reviewing component patterns or form/mutation code |
| `references/performance.md` | React Compiler setup, when to drop manual memoization, when to keep it, Rules of React compliance | When reviewing performance-sensitive code or Compiler setup |
| `references/typescript.md` | ref-as-prop typing, `useRef` argument, `ReactElement.props`, JSX namespace scoping, `useReducer`, `useActionState` generics, ref callback return | When reviewing TypeScript types or type errors |
| `references/testing.md` | `act` import, async test patterns, testing Actions and optimistic UI, RTL v16 patterns | When reviewing or writing tests |
| `references/migration.md` | Removed APIs, deprecations, codemods, step-by-step upgrade guide | When upgrading from React 18 or auditing for removed APIs |

## Output Format

For each finding:

1. **File and line(s)** affected
2. **Rule violated** (e.g. "useFormStatus must be called inside a child component, not the form itself")
3. **Severity:** Critical / High / Medium / Low
4. **Before/after code snippet**

End with a summary table of all findings sorted by severity.
