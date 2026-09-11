# React Native — Testing

Version target: React Native 0.76+ / Jest 29+ / RNTL 12+ / Detox 20+ / Maestro

---

## Testing Pyramid

| Layer | Tool | Speed | Confidence | Use for |
|-------|------|-------|-----------|---------|
| Static analysis | ESLint + TypeScript | Instant | Low | Type errors, lint violations |
| Unit tests | Jest | Fast | Medium | Business logic, pure functions, hooks |
| Component tests | RNTL | Fast | High | UI interactions, rendering, accessibility |
| Integration tests | RNTL + mocked services | Medium | High | Feature flows without device |
| E2E tests | Detox or Maestro | Slow | Very High | Auth flow, payments, critical journeys |

---

## Jest Configuration

### `jest.config.js` (Expo projects)

```js
module.exports = {
  preset: 'jest-expo',
  setupFilesAfterFramework: ['@testing-library/jest-native/extend-expect'],
  transformIgnorePatterns: [
    'node_modules/(?!((jest-)?react-native|@react-native(-community)?)|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@unimodules/.*|unimodules|sentry-expo|native-base|react-native-svg)',
  ],
};
```

### `jest.config.js` (bare React Native)

```js
module.exports = {
  preset: 'react-native',
  setupFilesAfterFramework: ['@testing-library/jest-native/extend-expect'],
};
```

**Rules:**
- Use `jest-expo` preset for Expo projects.
- `transformIgnorePatterns` regex must include all packages that ship untransformed ESM.
- Always import `@testing-library/jest-native/extend-expect` to unlock `.toBeVisible()`, `.toHaveTextContent()`, etc.

---

## React Native Testing Library (RNTL)

### Basic render test

```tsx
import { render, screen } from '@testing-library/react-native';
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
  it('renders email and password fields', () => {
    render(<LoginForm onSubmit={jest.fn()} />);
    expect(screen.getByLabelText('Email')).toBeVisible();
    expect(screen.getByLabelText('Password')).toBeVisible();
  });
});
```

### Interaction test

```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';

it('calls onSubmit with credentials when form is submitted', async () => {
  const onSubmit = jest.fn();
  render(<LoginForm onSubmit={onSubmit} />);

  fireEvent.changeText(screen.getByLabelText('Email'), 'user@example.com');
  fireEvent.changeText(screen.getByLabelText('Password'), 'secret123');
  fireEvent.press(screen.getByRole('button', { name: 'Log in' }));

  await waitFor(() => {
    expect(onSubmit).toHaveBeenCalledWith({
      email: 'user@example.com',
      password: 'secret123',
    });
  });
});
```

### Query priority (prefer in this order)

| Query | When to use |
|-------|------------|
| `getByRole` | Buttons, inputs, headings — most like a user |
| `getByLabelText` | Form fields with accessible labels |
| `getByPlaceholderText` | Input fields (less preferred) |
| `getByText` | Static text content |
| `getByDisplayValue` | Current value of input |
| `getByTestId` | Last resort — couples tests to implementation |

**Rules:**
- Never use `getByTestId` as the primary query.
- Prefer `screen` queries over destructured render queries.
- Use `waitFor` for any assertion that depends on async state updates.
- Don't test implementation details (internal state, hook calls) — test what the user sees.
- Avoid large snapshot tests — they break on trivial changes and don't express intent.

### Mocking navigation (React Navigation)

```tsx
import { NavigationContainer } from '@react-navigation/native';

const Wrapper = ({ children }: { children: React.ReactNode }) => (
  <NavigationContainer>{children}</NavigationContainer>
);

render(<ScreenComponent />, { wrapper: Wrapper });
```

### Mocking navigation (Expo Router)

```tsx
import { renderRouter, screen } from 'expo-router/testing-library';

it('navigates to product detail on press', async () => {
  renderRouter({
    index: () => <HomeScreen />,
    'product/[id]': () => <ProductScreen />,
  });

  fireEvent.press(screen.getByText('View Product'));
  expect(await screen.findByText('Product Details')).toBeVisible();
});
```

---

## Mocking Native Modules

### Manual mock — `__mocks__` directory

```tsx
// __mocks__/@react-native-async-storage/async-storage.ts
const AsyncStorage = {
  getItem: jest.fn(),
  setItem: jest.fn(),
  removeItem: jest.fn(),
  clear: jest.fn(),
  getAllKeys: jest.fn(),
};
export default AsyncStorage;
```

### Inline mock

```tsx
jest.mock('react-native/Libraries/Utilities/Platform', () => ({
  OS: 'android',
  Version: 30,
  select: jest.fn(obj => obj.android ?? obj.default),
}));
```

**Rules:**
- Mock at the boundary — mock the module interface, not internal implementation.
- Restore mocks between tests with `restoreMocks: true` in Jest config or `afterEach`.

---

## Async Testing Patterns

```tsx
// ✅ waitFor — retries until it passes or times out
await waitFor(() => expect(screen.getByText('Loaded!')).toBeVisible());

// ✅ findBy* — shorthand for getBy* + waitFor
const loaded = await screen.findByText('Loaded!');

// ✅ userEvent (RNTL v12+ — more realistic simulation)
import userEvent from '@testing-library/user-event';
const user = userEvent.setup();
await user.type(screen.getByLabelText('Email'), 'test@example.com');
await user.press(screen.getByRole('button', { name: 'Submit' }));

// ❌ Don't use arbitrary timeouts
await new Promise(r => setTimeout(r, 500)); // flaky
```

---

## E2E Testing

### Detox (recommended for CI with simulators/emulators)

```ts
// e2e/login.test.ts
describe('Login flow', () => {
  beforeAll(async () => { await device.launchApp(); });
  beforeEach(async () => { await device.reloadReactNative(); });

  it('logs in with valid credentials', async () => {
    await element(by.label('Email')).typeText('user@example.com');
    await element(by.label('Password')).typeText('secret123');
    await element(by.label('Log in')).tap();
    await expect(element(by.text('Welcome back!'))).toBeVisible();
  });
});
```

### Maestro (simpler YAML-based flows)

```yaml
# .maestro/login.yaml
appId: com.example.myapp
---
- launchApp
- tapOn:
    id: "email-input"
- inputText: "user@example.com"
- tapOn:
    id: "password-input"
- inputText: "secret123"
- tapOn:
    text: "Log in"
- assertVisible:
    text: "Welcome back!"
```

**Rules:**
- Cover critical user paths with E2E: auth, payments, core feature flows.
- Run E2E tests against release builds, not debug builds.
- Each test must set up its own state — no shared state between tests.
- Prefer Detox for existing iOS/Android CI; Maestro for quick, readable flows without a build step.

---

## Testing Checklist

- [ ] Unit tests for all pure business logic functions
- [ ] Component tests use `getByRole` / `getByLabelText` as primary queries
- [ ] Async assertions use `waitFor` or `findBy*`
- [ ] Native modules are mocked at the module boundary
- [ ] No `testID` as primary query (last resort only)
- [ ] No large snapshot tests
- [ ] E2E tests cover auth flow and at least one critical user journey
- [ ] CI runs unit + component tests on every PR
- [ ] CI runs E2E tests on release builds before merge to main
