# React 19 Migration — From React 18

Removed APIs, deprecations, and step-by-step upgrade guide.

---

## Removed React APIs

| Removed | Replacement |
|---------|-------------|
| `propTypes` checks | TypeScript types |
| `defaultProps` on function components | ES6 default parameters |
| Legacy context (`contextTypes`, `getChildContext`) | `createContext` + `useContext` |
| String refs (`ref="myRef"`) | Callback refs or `useRef` |
| `React.createFactory` | JSX |
| `react-test-renderer/shallow` | `npm install react-shallow-renderer` |

**`defaultProps` migration:**
```tsx
// ❌ Before
function Heading({ text }: { text: string }) { return <h1>{text}</h1>; }
Heading.defaultProps = { text: 'Hello' };

// ✅ After
function Heading({ text = 'Hello' }: { text?: string }) { return <h1>{text}</h1>; }
```

Note: `defaultProps` on **class components** still works — no ES6 equivalent exists.

---

## Removed React DOM APIs

| Removed | Replacement |
|---------|-------------|
| `ReactDOM.render(el, container)` | `createRoot(container).render(el)` |
| `ReactDOM.hydrate(el, container)` | `hydrateRoot(container, el)` |
| `ReactDOM.unmountComponentAtNode(container)` | `root.unmount()` |
| `ReactDOM.findDOMNode(instance)` | DOM refs (`useRef`) |
| `react-dom/test-utils` (all exports) | `@testing-library/react`; `act` from `react` |

```tsx
// ❌ React 18
import { render } from 'react-dom';
render(<App />, document.getElementById('root'));

// ✅ React 19
import { createRoot } from 'react-dom/client';
createRoot(document.getElementById('root')!).render(<App />);
```

---

## New Error Handling APIs on `createRoot`

```tsx
const root = createRoot(container, {
  onUncaughtError(error, errorInfo) {
    // Not caught by any Error Boundary → report to telemetry
    reportToSentry(error);
  },
  onCaughtError(error, errorInfo) {
    // Caught by an Error Boundary → log as warning
    console.warn(error);
  },
  onRecoverableError(error, errorInfo) {
    // React auto-recovered → optional logging
  },
});
```

---

## Deprecations (not yet removed — remove proactively)

| Deprecated | Replacement |
|------------|-------------|
| `<Context.Provider>` | `<Context value={...}>` |
| `forwardRef` | ref as a regular prop |
| `element.ref` access | `element.props.ref` |
| `react-test-renderer` | `@testing-library/react` |
| `MutableRefObject` | `RefObject` |

---

## Other Breaking Changes

| Change | Detail |
|--------|--------|
| New JSX Transform required | Set `"jsx": "react-jsx"` in tsconfig; old transform (`React` in scope) no longer supported |
| UMD builds removed | Use ESM CDN (`esm.sh`) for script-tag usage |
| JavaScript URLs rejected | `href="javascript:void(0)"` now throws an error |
| Empty string `src`/`href` | Warns and is not set (except on `<a>`) |

---

## Step-by-Step Upgrade Guide

**Step 1 — Upgrade to React 18.3 first**

React 18.3 is identical to 18.2 but adds deprecation warnings for all APIs removed in 19:
```bash
npm install react@18.3 react-dom@18.3 @types/react@18.3 @types/react-dom@18.3
```
Fix all warnings before continuing.

**Step 2 — Run the React 19 migration codemod**

```bash
# Runs all JS/TSX codemods in one shot
npx codemod@latest react/19/migration-recipe
```

Individual codemods (if you need selective application):

| Codemod | What it fixes |
|---------|---------------|
| `react/19/replace-reactdom-render` | `ReactDOM.render` → `createRoot`, `hydrate` → `hydrateRoot`, `unmountComponentAtNode` → `root.unmount()` |
| `react/19/replace-string-ref` | String refs → callback refs |
| `react/19/replace-act-import` | `react-dom/test-utils` `act` → `react` `act` |
| `react/19/replace-use-form-state` | `ReactDOM.useFormState` → `useActionState` |
| `react/prop-types-typescript` | `propTypes` → TypeScript types |

**Step 3 — Run TypeScript codemods**

```bash
npx types-react-codemod@latest preset-19 ./src
```

Individual TypeScript codemods:

| Codemod | What it fixes |
|---------|---------------|
| `no-implicit-ref-callback-return` | Implicit ref callback returns → explicit block bodies |
| `refobject-defaults` | `useRef()` → `useRef(undefined)` |
| `scoped-jsx` | Global `JSX` namespace → `declare module "react/jsx-runtime"` |
| `react-element-default-any-props` | Untyped `element.props` access |

**Step 4 — Install React 19**

```bash
npm install react@^19.0.0 react-dom@^19.0.0
npm install -D @types/react@^19.0.0 @types/react-dom@^19.0.0
```

**Step 5 — Ensure new JSX Transform**

`tsconfig.json`:
```json
{ "compilerOptions": { "jsx": "react-jsx" } }
```

**Step 6 — Run build and tests**

```bash
npm run build
npm run test:run
```

Fix any remaining TypeScript errors (see `references/typescript.md`).
