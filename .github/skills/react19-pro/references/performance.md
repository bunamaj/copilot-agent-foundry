# React 19 Performance & React Compiler

> **Status in this workspace:** React Compiler is **NOT currently installed** (`vite.config.ts` has no `babel-plugin-react-compiler` entry). Until installed, treat the "Compiler handles automatically" guidance below as aspirational — manual `useMemo`, `useCallback`, and `React.memo` rules still apply in full.

---

## React Compiler (stable as of 2025-10-07)

React Compiler is a **build-time tool** that automatically inserts memoization, eliminating the need to manually write `useMemo`, `useCallback`, and `React.memo` in most cases.

### Installation (Vite — this workspace)

```bash
npm install -D babel-plugin-react-compiler@latest eslint-plugin-react-compiler@latest
```

**vite.config.ts:**
```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [
    react({
      babel: {
        plugins: [['babel-plugin-react-compiler', {}]],
      },
    }),
  ],
});
```

---

## What the Compiler Handles Automatically

With React Compiler enabled, **do not** manually write:
- `React.memo(Component)` — component memoization is automatic
- `useMemo(() => expensiveCalc(), [deps])` — pure computations are memoized
- `useCallback(() => fn(), [deps])` — function stability is handled

**Before (manual — error-prone):**
```tsx
const ExpensiveList = memo(function ExpensiveList({ data, onClick }) {
  const processed = useMemo(() => transform(data), [data]);
  const handleClick = useCallback((item) => onClick(item.id), [onClick]);
  return (
    <ul>
      {processed.map(item => (
        // 🐛 Bug: arrow function creates new ref every render, breaks memo
        <Item key={item.id} onClick={() => handleClick(item)} />
      ))}
    </ul>
  );
});
```

**After (compiler handles it):**
```tsx
function ExpensiveList({ data, onClick }) {
  const processed = transform(data);
  const handleClick = (item: Item) => onClick(item.id);
  return (
    <ul>
      {processed.map(item => (
        <Item key={item.id} onClick={() => handleClick(item)} />
      ))}
    </ul>
  );
}
```

---

## When Manual Memoization Still Makes Sense

Even with React Compiler, these cases justify manual hooks:

| Scenario | Recommendation |
|----------|----------------|
| A memoized value is used as an effect dependency and you need to suppress re-firing | `useMemo` as an explicit escape hatch |
| Stabilizing a callback passed to a third-party native event subscription | `useCallback` |
| You need to guarantee a specific memoization boundary the compiler may not infer | `useMemo` / `useCallback` |
| Existing code with `useMemo`/`useCallback` | Leave it — removing can change compilation output |

**Official guidance:** For new code, rely on the compiler. Use `useMemo`/`useCallback` where needed for precise control.

---

## Rules of React — Required for Compiler Correctness

The compiler **requires** components to follow the [Rules of React](https://react.dev/reference/rules). Violations cause the compiler to skip optimization for that component.

**Critical rules:**

```tsx
// 🚩 Mutating props — violates purity
function Bad({ items }: { items: Item[] }) {
  items.push({ id: 'new' }); // ❌ mutates prop directly
  return <List items={items} />;
}

// ✅ Pure — return new value
function Good({ items }: { items: Item[] }) {
  const withNew = [...items, { id: 'new' }];
  return <List items={withNew} />;
}
```

```tsx
// 🚩 Mutating state directly
function Bad() {
  const [items, setItems] = useState<Item[]>([]);
  items.push({ id: 'new' }); // ❌ mutates state directly
}

// ✅ Correct
function Good() {
  const [items, setItems] = useState<Item[]>([]);
  setItems(prev => [...prev, { id: 'new' }]); // ✅
}
```

### ESLint Plugin (catches violations)

Install even without the compiler — prevents regressions:

```bash
npm install -D eslint-plugin-react-compiler@latest
```

```js
// eslint.config.js
import reactCompiler from 'eslint-plugin-react-compiler';
export default [
  { plugins: { 'react-compiler': reactCompiler } },
  { rules: { 'react-compiler/react-compiler': 'error' } },
];
```

---

## `useDeferredValue` — New `initialValue` Parameter

```tsx
function Search({ query }: { query: string }) {
  // ✅ New in React 19: provide an initialValue for the first render
  const deferredQuery = useDeferredValue(query, '');
  // On first render: uses '' immediately, then schedules a background re-render with query
  return <Results query={deferredQuery} />;
}
```

Without `initialValue` (React 18 behavior): initial render uses the initial `query` value. With `initialValue`: initial render uses `''`, which is often better for perceived performance (avoid expensive render on first paint).
