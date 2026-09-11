# React 19 TypeScript Changes

Breaking and changed type patterns in React 19.x `@types/react@^19`.

---

## `ref` as Prop — New Typing

```tsx
import { useRef, type Ref } from 'react';

// ✅ React 19: ref is a normal prop with a normal type
interface MyInputProps {
  placeholder: string;
  ref?: Ref<HTMLInputElement>;
}

function MyInput({ placeholder, ref }: MyInputProps) {
  return <input placeholder={placeholder} ref={ref} />;
}

// Calling code — unchanged
const inputRef = useRef<HTMLInputElement>(null);
<MyInput placeholder="Enter text" ref={inputRef} />
```

**`forwardRef` is deprecated** — still works in 19.x but will be removed. See `references/migration.md` for the codemod.

---

## `useRef` Requires an Argument

```ts
// ❌ TypeScript error in React 19 — useRef requires one argument
const ref = useRef();

// ✅ Correct
const ref = useRef(undefined);            // RefObject<undefined>
const ref = useRef<number>(null);         // RefObject<number | null>
const ref = useRef<HTMLDivElement>(null); // RefObject<HTMLDivElement | null>
```

**`MutableRefObject` is deprecated** — the type is now unified into a single `RefObject<T>`:

```ts
// Deprecated — do not use for new code
const ref: React.MutableRefObject<number> = useRef(0);

// ✅ Unified type
const ref: React.RefObject<number> = useRef(0);
```

---

## Ref Callback Must Return `void` or a Cleanup Function

```ts
// ❌ TypeScript error — implicit return of HTMLDivElement
<div ref={current => (instance = current)} />

// ✅ Explicit block body (returns void)
<div ref={current => { instance = current; }} />

// ✅ Cleanup return
<div ref={current => {
  instance = current;
  return () => { instance = null; };
}} />
```

---

## `useActionState` Generics

Type is fully inferred from the action function signature — no need to provide generics explicitly:

```ts
interface FormState { error: string | null; count: number; }

async function myAction(prevState: FormState, payload: FormData): Promise<FormState> {
  return { error: null, count: prevState.count + 1 };
}

// ✅ All types inferred
const [state, dispatch, isPending] = useActionState(myAction, { error: null, count: 0 });
//    ^FormState            ^(FormData)=>void   ^boolean
```

---

## `useReducer` Typings — Rely on Inference

```ts
// ❌ Old — passing full Reducer type explicitly
useReducer<React.Reducer<State, Action>>(reducer);

// ✅ React 19 — rely on inference
useReducer(reducer);

// If explicit types are needed, annotate the function parameters
useReducer((state: State, action: Action) => state);

// Or use the new tuple form
useReducer<State, [Action]>(reducer);
```

---

## `ReactElement.props` is Now `unknown` (was `any`)

```ts
// Before React 19: element.props was `any` — accidental access never errored
const before: ReactElement['props']; // any

// React 19: element.props is `unknown` — you must narrow before accessing
const after: ReactElement['props']; // unknown

// ✅ Correct — use explicit type param to keep typed access
const el: ReactElement<{ id: string }> = <MyComponent id="abc" />;
el.props.id; // string ✓

// ✅ Or use a type assertion (only when you control the shape)
const props = (element as ReactElement<{ id: string }>).props;
```

If you have existing code with untyped `element.props` access, run the codemod:
```bash
npx types-react-codemod@latest react-element-default-any-props ./src
```

---

## JSX Namespace — Must Be Scoped

```ts
// ❌ Old — global JSX namespace (breaks in strict tsconfig, removed in @types/react@19)
declare namespace JSX {
  interface IntrinsicElements { 'my-element': { myProp: string }; }
}

// ✅ React 19 — scoped inside declare module
// The correct module depends on your tsconfig "jsx" setting:
//   "react-jsx"    → "react/jsx-runtime"
//   "react-jsxdev" → "react/jsx-dev-runtime"
//   "react"        → "react"

declare module 'react/jsx-runtime' {
  namespace JSX {
    interface IntrinsicElements {
      'my-element': { myProp: string };
    }
  }
}
```

Run the codemod to fix automatically:
```bash
npx types-react-codemod@latest scoped-jsx ./src
```

---

## `useRef` Codemod

Fix `useRef()` calls missing arguments:
```bash
npx types-react-codemod@latest refobject-defaults ./src
```

---

## Summary of Type Breaking Changes

| Change | Old | New | Fix |
|--------|-----|-----|-----|
| `useRef()` requires arg | `useRef()` | `useRef(undefined)` | `refobject-defaults` codemod |
| `ReactElement.props` | `any` | `unknown` | `react-element-default-any-props` codemod |
| `JSX` namespace | global | scoped module | `scoped-jsx` codemod |
| Ref callback return | implicit return ok | must return `void` or cleanup | `no-implicit-ref-callback-return` codemod |
| `MutableRefObject` | used | deprecated | use `RefObject<T>` |
