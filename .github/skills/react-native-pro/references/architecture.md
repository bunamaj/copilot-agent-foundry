# React Native — New Architecture

React Native 0.76+ ships with the **New Architecture enabled by default**. Every new project should use it; legacy bridge opt-out requires explicit configuration.

---

## Key Components of the New Architecture

| Component | Role |
|-----------|------|
| **Fabric** | New C++ rendering engine. Synchronous layout, proper concurrent React support. |
| **JSI (JavaScript Interface)** | Replaces the async JSON bridge. JS holds direct C++ object references; method calls have no serialization cost. |
| **TurboModules** | Lazily-loaded native modules registered via JSI. Replace legacy `NativeModules` bridge. |
| **Codegen** | Generates type-safe C++/Java/Obj-C bindings from TypeScript specs. Eliminates runtime type mismatches. |
| **Hermes** | Default JS engine (on by default since RN 0.70). Enables bytecode compilation at build time. |
| **Bridgeless mode** | Removes the legacy MessageQueue bridge entirely (enabled by default in 0.76). |

---

## New Architecture Enablement

### Android — `android/gradle.properties`
```properties
# New Architecture is ON by default in RN 0.76+
# Only set to false if you have a hard blocker:
newArchEnabled=true
```

### iOS — `ios/Podfile`
```ruby
# Do NOT set RCT_NEW_ARCH_ENABLED=0 unless you have a specific reason.
# The value defaults to 1 in RN 0.76+.
```

**Anti-pattern:** Disabling the New Architecture on new projects without a documented reason — every popular library supports it since late 2024.

---

## Hermes

- Default JS engine on Android (since RN 0.70) and iOS (since RN 0.64, default 0.70).
- Compiles JS to bytecode at **build time**, not at startup — reduces cold start time.
- Supports modern JS (ES2022, async/await, optional chaining, nullish coalescing).
- Profile with Chrome DevTools via Hermes Sampling Profiler in the Dev Menu.

**Rules:**
- Never disable Hermes without a concrete benchmark showing it hurts your app.
- Avoid `eval()` — Hermes strips it at build time; it will throw at runtime.
- `Function.prototype.toString()` is not supported; don't rely on it for serialization.

---

## TurboModules — Writing a Native Module

### TypeScript Spec (Codegen entry point)

```ts
// NativeMyModule.ts  — must live in the package root
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  readonly multiply: (a: number, b: number) => number;
}

export default TurboModuleRegistry.strictGet<Spec>('MyModule');
```

**Rules:**
- The spec file **must** be named `Native<ModuleName>.ts` for Codegen to pick it up.
- All types must be expressible in the RN type system: primitives, `Object`, arrays, `Promise<T>`, and `RootTag`.
- Use `TurboModuleRegistry.strictGet` (throws if module not found) for required modules; use `get` (returns null) for optional ones.
- Never mix TurboModule and legacy `NativeModules` access for the same module.

### Android Registration — `ReactPackage`

```kotlin
class MyPackage : TurboReactPackage() {
  override fun getModule(name: String, context: ReactApplicationContext): NativeModule? =
    if (name == MyModule.NAME) MyModule(context) else null

  override fun getReactModuleInfoProvider() = ReactModuleInfoProvider {
    mapOf(MyModule.NAME to ReactModuleInfo(MyModule.NAME, MyModule.NAME,
      false, false, false, BuildConfig.IS_NEW_ARCHITECTURE_ENABLED))
  }
}
```

### iOS Registration

Codegen wires this automatically in RN 0.73+ with the new template. No manual registration needed for TurboModules when using Codegen.

---

## Fabric — Custom Native Components

### TypeScript Spec

```ts
// NativeMyComponentNativeComponent.ts
import type { ViewProps } from 'react-native';
import type { HostComponent } from 'react-native';
import codegenNativeComponent from 'react-native/Libraries/Utilities/codegenNativeComponent';

interface NativeProps extends ViewProps {
  color?: string;
}

export default codegenNativeComponent<NativeProps>('MyComponent') as HostComponent<NativeProps>;
```

**Rules:**
- Custom Fabric components must extend `ViewProps`.
- Event handlers must follow the `onEventName` convention and use `DirectEventHandler` or `BubblingEventHandler` types.
- Never use `findNodeHandle` — it is a legacy API. Use `ref` callbacks instead.

---

## Concurrent React Features in React Native

Enabled by Fabric (New Architecture):

| Feature | Status | Notes |
|---------|--------|-------|
| `useTransition` | ✅ Supported | Mark low-priority state updates |
| `useDeferredValue` | ✅ Supported | Defer expensive re-renders |
| `Suspense` for data | ✅ Supported | Pair with a Suspense-aware data library |
| Automatic batching | ✅ Default | Multiple `setState` calls in async code batch automatically |
| Synchronous layout effects | ✅ via Fabric | `useLayoutEffect` runs before paint |

```tsx
// ✅ Use startTransition for non-urgent list filters
import { useTransition, useState } from 'react';
import { TextInput } from 'react-native';

function SearchResults() {
  const [isPending, startTransition] = useTransition();
  const [query, setQuery] = useState('');

  return (
    <TextInput
      onChangeText={text => {
        startTransition(() => setQuery(text)); // non-urgent
      }}
    />
  );
}
```

---

## Common New Architecture Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| `NativeModules.MyModule.method()` (legacy bridge) | Use TurboModule spec + `TurboModuleRegistry.get` |
| `requireNativeComponent('MyView')` (legacy Fabric) | Use `codegenNativeComponent` spec |
| `findNodeHandle(ref)` | Use `ref.current?.measure(...)` directly |
| Disabling `newArchEnabled` globally | Only opt-out at the module level if a dependency is broken |
| Creating a Promise in a TurboModule without resolving/rejecting on all paths | Always resolve or reject in a finally block |
