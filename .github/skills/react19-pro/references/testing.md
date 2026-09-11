# React 19 Testing

Testing patterns for React 19 with Vitest + `@testing-library/react` v16.

---

## `act` Import Change — Critical

```ts
// ❌ Old — react-dom/test-utils is completely removed in React 19
import { act } from 'react-dom/test-utils';

// ✅ React 19
import { act } from 'react';
```

Run the codemod to fix automatically:
```bash
npx codemod@latest react/19/replace-act-import
```

---

## Always Use `await act(async ...)` 

The synchronous form of `act` is unreliable with concurrent features. React plans to deprecate and remove the sync form.

```ts
// ❌ Sync act — unreliable in React 19 concurrent mode
act(() => { root.render(<MyComponent />); });

// ✅ Async act — always prefer
await act(async () => { root.render(<MyComponent />); });
```

---

## RTL v16 — `act` Is Automatic

`@testing-library/react` v16 wraps all its helpers in `act` internally. You rarely need to call `act` directly:

```tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

test('submits form', async () => {
  const user = userEvent.setup();
  render(<ProfileForm />);

  await user.type(screen.getByLabelText('Name'), 'Alice');
  await user.click(screen.getByRole('button', { name: /save/i }));

  // RTL handles all act() wrapping internally
  await waitFor(() => {
    expect(screen.getByText('Saved!')).toBeInTheDocument();
  });
});
```

---

## Testing `useActionState` — Pending State + Outcome

```tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { vi } from 'vitest';

vi.mock('./actions', () => ({
  updateProfile: vi.fn().mockResolvedValue(undefined),
}));

test('shows pending state then success', async () => {
  const user = userEvent.setup();
  render(<ProfileForm />);

  expect(screen.getByRole('button')).toHaveTextContent('Save');

  await user.click(screen.getByRole('button', { name: /save/i }));

  // Pending state — use waitFor because RTL's act() may flush the
  // microtask queue before this assertion runs, causing a false pass
  await waitFor(() => {
    expect(screen.getByRole('button')).toBeDisabled();
  });

  // Settled state
  await waitFor(() => {
    expect(screen.getByText('Saved!')).toBeInTheDocument();
  });
});

test('shows error state on failure', async () => {
  const { updateProfile } = await import('./actions');
  vi.mocked(updateProfile).mockRejectedValueOnce(new Error('Network error'));

  const user = userEvent.setup();
  render(<ProfileForm />);

  await user.click(screen.getByRole('button', { name: /save/i }));

  await waitFor(() => {
    expect(screen.getByText('Failed to save')).toBeInTheDocument();
  });
});
```

---

## Testing `useOptimistic` — Immediate UI + Server Confirmation

```tsx
test('shows optimistic update immediately', async () => {
  const user = userEvent.setup();
  // Slow mock — simulates in-flight request
  const mockToggle = vi.fn(() => new Promise<void>(res => setTimeout(res, 500)));

  render(<LikeButton isLiked={false} onToggle={mockToggle} />);
  expect(screen.getByRole('button')).toHaveTextContent('🤍 Like');

  await user.click(screen.getByRole('button'));

  // Optimistic update appears immediately (before server responds)
  expect(screen.getByRole('button')).toHaveTextContent('❤️ Unlike');

  // After server responds, state is confirmed
  await waitFor(() => {
    expect(screen.getByRole('button')).toHaveTextContent('❤️ Unlike');
  });
});
```

---

## Testing Async Transitions

```tsx
test('button is disabled while transition is pending', async () => {
  const user = userEvent.setup();
  const mockSave = vi.fn(() => new Promise<void>(res => setTimeout(res, 100)));

  render(<SaveButton onSave={mockSave} />);
  expect(screen.getByRole('button')).toHaveTextContent('Save');

  await user.click(screen.getByRole('button'));

  // Pending state — button disabled
  expect(screen.getByRole('button')).toBeDisabled();
  expect(screen.getByRole('button')).toHaveTextContent('Saving…');

  // Resolved state
  await waitFor(() => {
    expect(screen.getByRole('button')).not.toBeDisabled();
  });
});
```

---

## `IS_REACT_ACT_ENVIRONMENT`

Vitest and RTL set this automatically. If using a custom test environment, add to global setup:

```ts
// tests/setup.ts
(global as unknown as Record<string, unknown>).IS_REACT_ACT_ENVIRONMENT = true;
```

---

## `react-test-renderer` is Deprecated

`react-test-renderer` logs a deprecation warning in React 19 and switches to concurrent rendering by default. Migrate to `@testing-library/react`:

```ts
// ❌ Deprecated
import { create } from 'react-test-renderer';

// ✅ Use RTL
import { render } from '@testing-library/react';
```

---

## Vitest + MSW (this workspace)

This workspace uses Vitest + MSW (`msw@^2`). Ensure MSW handlers are active before rendering:

```ts
// tests/setup.ts — already configured in this workspace
import { server } from './mocks/server';
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

For overriding a handler in a specific test:
```ts
import { http, HttpResponse } from 'msw';

test('handles server error', async () => {
  server.use(
    http.post('/api/profile', () => HttpResponse.json({ error: 'Server error' }, { status: 500 }))
  );
  // ... render and assert
});
```
