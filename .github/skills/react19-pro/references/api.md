# React 19 New APIs

New APIs added in React 19.x and their correct usage.

---

## `use(resource)` — React API, not a hook

**Package:** `react` | **Can be called conditionally: yes**

```ts
function use<T>(resource: Promise<T> | Context<T>): T
```

**Reading a Promise (Suspense integration):**
```tsx
// ✅ Pass a stable promise created outside render (e.g. from a Server Component)
function Comments({ commentsPromise }: { commentsPromise: Promise<Comment[]> }) {
  const comments = use(commentsPromise); // suspends; nearest <Suspense> shows fallback
  return comments.map(c => <p key={c.id}>{c.body}</p>);
}
```

**Reading Context conditionally (impossible with `useContext`):**
```tsx
function Heading({ children }: { children?: React.ReactNode }) {
  if (!children) return null;        // early return before use() — allowed
  const theme = use(ThemeContext);   // ✅ conditional context read
  return <h1 style={{ color: theme === 'dark' ? 'white' : 'black' }}>{children}</h1>;
}
```

**Error handling — cannot use try/catch; use `.catch()` or Error Boundary:**
```tsx
// ✅ Option 1: Error Boundary wrapping Suspense
<ErrorBoundary fallback={<p>Error</p>}><Suspense fallback={<p>Loading</p>}><Comp promise={p} /></Suspense></ErrorBoundary>

// ✅ Option 2: .catch() on the promise before passing it
const safe = fetchData().catch(() => 'fallback');
const data = use(safe);
```

**Rules:**
- Must be called inside a component or hook, not a plain utility function.
- **Never** create a promise inside render and pass it to `use` — React warns: *"A component was suspended by an uncached promise."* Create promises outside render or memoize with `useMemo`.
- Cannot be called inside `try/catch`.

| ✅ Use `use()` | ❌ Don't use `use()` |
|---|---|
| Reading context conditionally / after early returns | When `useContext` suffices (top-level, unconditional) |
| Consuming a Promise passed from a Server Component | Creating and consuming a promise in the same component |
| Inside loops or conditionals | Inside `try-catch` blocks |

---

## `useActionState(action, initialState, permalink?)` — form and mutation state

**Package:** `react` | (previously `ReactDOM.useFormState` — renamed)

```ts
function useActionState<State, Payload>(
  action: (prevState: Awaited<State>, payload: Payload) => State | Promise<State>,
  initialState: Awaited<State>,
  permalink?: string
): [state: Awaited<State>, dispatch: (payload: Payload) => void, isPending: boolean]
```

```tsx
async function submitForm(prevError: string | null, formData: FormData): Promise<string | null> {
  const name = formData.get('name') as string;
  if (!name.trim()) return 'Name is required';
  await updateProfile({ name });
  return null;
}

function ProfileForm() {
  const [error, action, isPending] = useActionState(submitForm, null);
  return (
    <form action={action}>
      <input name="name" disabled={isPending} />
      <button type="submit" disabled={isPending}>{isPending ? 'Saving…' : 'Save'}</button>
      {error && <p className="error">{error}</p>}
    </form>
  );
}
```

**Rules:**
- `dispatch` must be called inside a `startTransition` or wired as a `<form action>` prop. Calling outside a transition in dev throws: *"An async function with useActionState was called outside of a transition."*
- **`formData` is the second argument** (not first) — `prevState` is always first.
- Actions are **queued sequentially** — each receives the prior result as `prevState`.
- The action is **NOT** double-invoked in Strict Mode (unlike `useReducer` reducers) — side effects are allowed.

| ✅ Use `useActionState` | ❌ Don't use `useActionState` |
|---|---|
| Form submissions with server mutations | Read-only queries without state updates |
| When you need `prevState` to compute next state | Simple fire-and-forget mutations |

---

## `useFormStatus()` — form submission context

**Package:** `react-dom`

```ts
function useFormStatus(): {
  pending: boolean;
  data: FormData | null;
  method: 'get' | 'post' | null;
  action: ((formData: FormData) => void) | null;
}
```

```tsx
// ✅ Must be a CHILD component, not the component rendering the <form>
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button type="submit" disabled={pending}>{pending ? 'Saving…' : 'Save'}</button>;
}

function MyForm() {
  return (
    <form action={myAction}>
      <input name="title" />
      <SubmitButton />  {/* ✅ reads the parent form's status */}
    </form>
  );
}
```

**Critical rule — the component calling `useFormStatus` must be INSIDE the `<form>`:**
```tsx
// 🚩 Wrong — tracking its own form, pending is never true
function Form() {
  const { pending } = useFormStatus();
  return <form action={submit}><button disabled={pending}>Save</button></form>;
}
```

---

## `useOptimistic(value, reducer?)` — immediate optimistic UI

**Package:** `react`

```ts
function useOptimistic<State, Action = State>(
  value: State,
  reducer?: (currentState: State, action: Action) => State
): [optimisticState: State, setOptimistic: (action: Action) => void]
```

**Simple toggle:**
```tsx
const [optimisticIsLiked, setOptimisticIsLiked] = useOptimistic(isLiked);

function handleClick() {
  startTransition(async () => {
    setOptimisticIsLiked(!optimisticIsLiked); // immediate UI
    await toggleLike(!optimisticIsLiked);     // server sync
  });
}
```

**List with reducer (safe for concurrent add/remove):**
```tsx
type Action = { type: 'add'; item: Item } | { type: 'remove'; id: string };

const [optimisticItems, dispatch] = useOptimistic(
  items,
  (current, action: Action) => {
    if (action.type === 'add') return [...current, { ...action.item, pending: true }];
    if (action.type === 'remove') return current.filter(i => i.id !== action.id);
    return current;
  }
);
```

**Rules:**
- Setter **must** be called inside `startTransition` or an Action prop. Outside a transition, the optimistic value immediately reverts.
- Cannot be called during render — only from event handlers, effects, or callbacks.
- Use a **reducer** (second argument) for complex/composite state to avoid stale state.
- Auto-reverts to the real `value` when the transition completes or errors.

---

## `cache(fn)` — Server Components only

> **Not applicable in this workspace** — this project is a Vite SPA with no Server Components. `cache()` requires a React Server Component environment (e.g. Next.js App Router). In Client Components, use `useMemo` instead.

**Package:** `react`

```ts
function cache<T extends (...args: unknown[]) => unknown>(fn: T): T
```

```ts
// ✅ Define in a shared module — call cache() once at module level
export const getUser = cache(async (id: string) => db.user.query(id));

// In multiple Server Components — only one DB call per request
const user1 = await getUser('abc'); // fetches
const user2 = await getUser('abc'); // cache hit
```

**Rules:**
- Scoped per **server request** — reset on every new request.
- **Never call `cache()` inside a component** — each render creates a new function with an empty cache.
- Cache lookup uses `Object.is` — use primitive args or same object references.
- Errors thrown by `fn` are cached — same error re-thrown on cache hits.
- **Not for Client Components** — use `useMemo` in Client Components instead.

| `cache` | `useMemo` | `memo` |
|---------|-----------|--------|
| Server Components, per-request | Client Components, per-render | Client Components, per-props-change |

---

## `ref` as Prop — replaces `forwardRef`

```tsx
// ✅ React 19: ref is a regular prop — no forwardRef wrapper needed
function MyInput({ placeholder, ref }: { placeholder: string; ref?: React.Ref<HTMLInputElement> }) {
  return <input placeholder={placeholder} ref={ref} />;
}

// ❌ forwardRef still works in 19.x but is deprecated — will be removed in a future version
const MyInput = forwardRef<HTMLInputElement, Props>(({ placeholder }, ref) => (
  <input placeholder={placeholder} ref={ref} />
));
```

A codemod is available: `npx codemod@latest react/19/migration-recipe`

---

## `<Context>` as Provider — replaces `<Context.Provider>`

```tsx
const ThemeContext = createContext('light');

// ✅ React 19: use the Context object itself as a JSX element
<ThemeContext value="dark">{children}</ThemeContext>

// ❌ Deprecated (still works, will be removed in a future version)
<ThemeContext.Provider value="dark">{children}</ThemeContext.Provider>
```

---

## Ref Cleanup Functions

React 19 adds cleanup function support to ref callbacks:

```tsx
// ✅ Return a cleanup function from the ref callback
<input ref={(node) => {
  const sub = subscribe(node);
  return () => sub.unsubscribe(); // called on unmount instead of ref(null)
}} />
```

**TypeScript:** Ref callbacks that return non-cleanup, non-void values are a TypeScript error in React 19. See `references/typescript.md`.
