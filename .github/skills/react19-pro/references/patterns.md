# React 19 Idiomatic Patterns

Correct patterns for async transitions, form Actions, optimistic UI, and data fetching.

---

## Async Transitions

Wrap any async side effect in `startTransition` to get a pending state without blocking the UI:

```tsx
function SaveButton({ onSave }: { onSave: () => Promise<void> }) {
  const [isPending, startTransition] = useTransition();

  return (
    <button
      disabled={isPending}
      onClick={() => startTransition(async () => { await onSave(); })}
    >
      {isPending ? 'Saving…' : 'Save'}
    </button>
  );
}
```

**Important:** `isPending` stays `true` for the full duration of the async operation. State updates _before_ the first `await` are batched as a transition. State updates _after_ an `await` require a nested `startTransition`.

---

## Form Handling with Actions

Full pattern for a mutation form using `useActionState`:

```tsx
import { useActionState } from 'react';

interface FormState { error: string | null; success: boolean; }

async function submitForm(prev: FormState, formData: FormData): Promise<FormState> {
  const name = formData.get('name') as string;
  if (!name.trim()) return { error: 'Name is required', success: false };
  try {
    await updateProfile({ name });
    return { error: null, success: true };
  } catch {
    return { error: 'Failed to save', success: false };
  }
}

function ProfileForm() {
  const [state, action, isPending] = useActionState(submitForm, { error: null, success: false });

  return (
    <form action={action}>
      <input name="name" disabled={isPending} />
      <button type="submit" disabled={isPending}>{isPending ? 'Saving…' : 'Save'}</button>
      {state.error && <p className="error">{state.error}</p>}
      {state.success && <p className="success">Saved!</p>}
    </form>
  );
}
```

**Anti-pattern — calling dispatch outside a transition:**
```tsx
// 🚩 Wrong — dispatch must be inside startTransition or form action
function BadForm() {
  const [state, dispatch] = useActionState(action, initialState);
  return <button onClick={() => dispatch(payload)}>Save</button>; // missing startTransition
}

// ✅ Correct
<button onClick={() => startTransition(() => dispatch(payload))}>Save</button>
```

---

## Optimistic UI with `useOptimistic`

Pattern for a list with add/remove and full type safety:

```tsx
import { useOptimistic, startTransition } from 'react';

interface Item { id: string; label: string; pending?: boolean; }
type OptimisticAction = { type: 'add'; item: Item } | { type: 'remove'; id: string };

function ItemList({ items, onAdd, onRemove }: {
  items: Item[];
  onAdd: (item: Item) => Promise<void>;
  onRemove: (id: string) => Promise<void>;
}) {
  const [optimisticItems, dispatch] = useOptimistic(
    items,
    (current, action: OptimisticAction) => {
      switch (action.type) {
        case 'add':    return [...current, { ...action.item, pending: true }];
        case 'remove': return current.filter(i => i.id !== action.id);
        default:       return current;
      }
    }
  );

  function handleAdd(label: string) {
    const item = { id: crypto.randomUUID(), label };
    startTransition(async () => {
      dispatch({ type: 'add', item });
      await onAdd(item);
    });
  }

  function handleRemove(id: string) {
    startTransition(async () => {
      dispatch({ type: 'remove', id });
      await onRemove(id);
    });
  }

  return (
    <ul>
      {optimisticItems.map(item => (
        <li key={item.id} style={{ opacity: item.pending ? 0.5 : 1 }}>
          {item.label}
          <button onClick={() => handleRemove(item.id)}>Remove</button>
        </li>
      ))}
      <button onClick={() => handleAdd('New Item')}>Add</button>
    </ul>
  );
}
```

**Detecting pending state:**
```tsx
// Option 1: value inequality
const isPending = optimisticValue !== realValue;

// Option 2: flag in reducer output (preferred for lists)
const [list] = useOptimistic(items, (s, newItem) => [...s, { ...newItem, isPending: true }]);
```

---

## Data Fetching with `use()` + Suspense

```tsx
// Server Component (or data layer) — stable promise, created once per request
export default function Page() {
  const dataPromise = loadData(); // created outside render
  return (
    <Suspense fallback={<Skeleton />}>
      <DataDisplay dataPromise={dataPromise} />
    </Suspense>
  );
}

// Client Component
'use client';
import { use } from 'react';

function DataDisplay({ dataPromise }: { dataPromise: Promise<Data> }) {
  const data = use(dataPromise); // suspends until resolved
  return <div>{data.title}</div>;
}
```

**Anti-pattern — creating a promise inside render:**
```tsx
// 🚩 Wrong — new promise every render triggers "uncached promise" warning
function Bad() {
  const data = use(fetchData()); // creates a new promise every render
}

// ✅ Correct — stable promise from outside render
function Parent() {
  const p = useMemo(() => fetchData(), []); // or pass from Server Component
  return <Suspense fallback={<Spinner />}><Child p={p} /></Suspense>;
}
function Child({ p }: { p: Promise<Data> }) {
  const data = use(p);
  return <div>{data.title}</div>;
}
```

---

## Composing `useActionState` + `useOptimistic`

The canonical pattern for immediate feedback + server confirmation:

```tsx
import { useActionState, useOptimistic, startTransition } from 'react';

async function saveCountAction(prevCount: number, _: FormData): Promise<number> {
  return await apiUpdateCount(prevCount + 1);
}

function Counter({ initialCount }: { initialCount: number }) {
  const [confirmedCount, submitAction, isPending] = useActionState(saveCountAction, initialCount);
  const [optimisticCount, setOptimisticCount] = useOptimistic(confirmedCount);

  function handleIncrement() {
    startTransition(() => {
      setOptimisticCount(c => c + 1); // immediate UI
      submitAction(new FormData());   // queued server sync
    });
  }

  return (
    <div>
      <p>Count: {optimisticCount} {isPending && '(syncing…)'}</p>
      <button onClick={handleIncrement}>+1</button>
    </div>
  );
}
```

---

## Context-as-Provider Pattern

```tsx
// ✅ React 19: use the Context object directly
const ThemeContext = createContext<'dark' | 'light'>('light');

function App({ children }: { children: React.ReactNode }) {
  return <ThemeContext value="dark">{children}</ThemeContext>;
}

// ❌ Deprecated (still works, remove during migration)
<ThemeContext.Provider value="dark">{children}</ThemeContext.Provider>
```

---

## Suspense Fallback Behavior (React 19)

React 19 **immediately commits** the Suspense fallback when a component suspends — it no longer waits for sibling trees to render first. Side effect: sibling trees still "pre-warm" (start executing) even while the fallback is shown.

**What this means for your code:**
- Fallback appears faster — no change needed.
- Data fetches in suspended siblings start eagerly — this is intentional (waterfall prevention).
- No code change required; awareness prevents surprise when profiling.
