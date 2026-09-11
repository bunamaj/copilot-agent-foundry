---
name: react-native-pro
description: >-
  Comprehensively reviews React Native code for best practices on cross-platform
  development, New Architecture (Fabric, JSI, TurboModules), Expo Router / React
  Navigation, core components, StyleSheet, platform-specific patterns, performance
  (FlatList, Animated, Reanimated), TypeScript, testing (Jest, RNTL, Detox), and
  native module authoring. Use when reading, writing, or reviewing React Native
  projects targeting Android, iOS, and web.
  Trigger keywords: React Native, Expo, Expo Router, React Navigation, Metro, Hermes,
  Fabric, TurboModules, JSI, NativeModule, NativeEventEmitter, FlatList, Animated,
  Reanimated, StyleSheet, Platform.OS, .android.ts, .ios.ts, Pressable, expo-router,
  create-expo-app, npx react-native, RN, RCT.
  DO NOT USE FOR: React (web-only), React 19 APIs (use react19-pro), React Native Web
  without a native target, or non-JavaScript mobile frameworks (Flutter, SwiftUI, Jetpack Compose).
---

Review React Native cross-platform code for correctness, idiomatic patterns, and New Architecture compliance. Version target: **React Native 0.76+ / Expo SDK 52+**.

## Review Process

1. **Load reference files** — identify which reference files apply to the change; read only those files.
2. **Check architecture** — verify New Architecture opt-in, JSI usage, TurboModule registration, and Hermes compatibility. See `references/architecture.md`.
3. **Check components** — verify core component usage, StyleSheet patterns, flex layout, accessibility props, and platform annotations. See `references/components.md`.
4. **Check cross-platform patterns** — verify `Platform.OS`, platform file extensions (`.ios.ts`, `.android.ts`, `.native.ts`), Expo Router file-based routes, and React Navigation navigator setup. See `references/cross-platform.md`.
5. **Check performance** — flag FlatList anti-patterns, anonymous `renderItem`, missing `keyExtractor`, undriven animations, and JS-thread-blocking patterns. See `references/performance.md`.
6. **Check testing** — verify Jest config, RNTL query strategy, mock patterns for native modules, Detox/Maestro E2E structure, and snapshot discipline. See `references/testing.md`.

## Reference Files

| File | Contents | When to load |
|------|----------|-------------|
| `references/architecture.md` | New Architecture (Fabric, JSI, TurboModules, Hermes), bridgeless mode, legacy bridge opt-out, Codegen, concurrent React features in RN | When reviewing native modules, performance-critical paths, or architecture configuration |
| `references/components.md` | Core components (View, Text, Image, TextInput, Pressable, FlatList, SectionList, ScrollView, Modal), StyleSheet API, flexbox, colors, accessibility | When reviewing UI components, styles, or accessibility |
| `references/cross-platform.md` | Platform module, file extensions, Expo Router file-based routing, React Navigation v7 (Stack, Tab, Drawer), deep linking, universal links, typed routes | When reviewing navigation setup, platform conditionals, or routing architecture |
| `references/performance.md` | JS vs UI thread, FlatList optimization props, Animated API (useNativeDriver), Reanimated 3, InteractionManager, LayoutAnimation, memo/useCallback patterns, Hermes profiling, FlashList | When reviewing list rendering, animations, or JS-thread-heavy code |
| `references/testing.md` | Jest + RN preset, React Native Testing Library (RNTL), mocking native modules, async patterns, snapshot discipline, Detox E2E, Maestro, CI setup | When reviewing or writing tests |

## Output Format

For each finding:

1. **File and line(s)** affected
2. **Rule violated** (e.g. "`renderItem` must be wrapped in `useCallback` to prevent recreation on each render")
3. **Severity:** Critical / High / Medium / Low
4. **Before/after code snippet**

End with a summary table of all findings sorted by severity.
