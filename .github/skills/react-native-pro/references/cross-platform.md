# React Native — Cross-Platform Patterns

Version target: React Native 0.76+ / Expo SDK 52+ / React Navigation 7 / Expo Router SDK 57

---

## Platform Detection

### `Platform` module

```tsx
import { Platform, StyleSheet } from 'react-native';

// Basic OS check
if (Platform.OS === 'ios') { /* ... */ }
if (Platform.OS === 'android') { /* ... */ }
if (Platform.OS === 'web') { /* ... */ }

// Version detection
const isAndroidNougat = Platform.OS === 'android' && Platform.Version >= 25;
const iOSMajor = Platform.OS === 'ios' ? parseInt(Platform.Version as string, 10) : 0;

// Platform.select — preferred for styles and values
const styles = StyleSheet.create({
  container: {
    paddingTop: Platform.select({ ios: 20, android: 0, default: 10 }),
  },
  shadow: Platform.select({
    ios: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 2 },
      shadowOpacity: 0.2,
      shadowRadius: 4,
    },
    android: {
      elevation: 4,
    },
    default: {},
  }),
});
```

**Rules:**
- Prefer `Platform.select` over `if (Platform.OS === ...)` in style definitions — it is more expressive and tree-shakeable.
- Always provide a `default` key in `Platform.select` when the value must exist on web or other platforms.

---

## Platform-Specific File Extensions

Metro resolves platform-specific extensions automatically:

```
Button.ios.tsx        ← loaded on iOS only
Button.android.tsx    ← loaded on Android only
Button.native.tsx     ← loaded on any native platform (iOS + Android), not web
Button.tsx            ← loaded on web (or as fallback)
```

**Import (no extension needed):**
```tsx
import Button from './Button'; // Metro picks the right file
```

**Rules:**
- Use platform files when component structure differs significantly, not just styles (use `Platform.select` for those).
- `.native.ts` is for shared native code that differs from the web version.
- Configure your web bundler to ignore `.native.js` extensions to avoid bundling native code for web.

---

## Expo Router (File-based routing — Recommended for new projects)

### Project structure

```
app/
  _layout.tsx          ← Root layout (NavigationContainer equivalent)
  index.tsx            ← "/" route (home screen)
  (tabs)/
    _layout.tsx        ← Tab navigator layout
    home.tsx           ← "/home" tab route
    profile.tsx        ← "/profile" tab route
  product/
    [id].tsx           ← "/product/:id" dynamic route
  +not-found.tsx       ← 404 fallback
```

### Root layout

```tsx
// app/_layout.tsx
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="index" options={{ title: 'Home' }} />
      <Stack.Screen name="product/[id]" options={{ title: 'Product' }} />
    </Stack>
  );
}
```

### Navigation

```tsx
import { router, useLocalSearchParams } from 'expo-router';

// Navigate
router.push('/product/123');
router.replace('/login');
router.back();

// Type-safe routes (enable in app.json: "experiments": { "typedRoutes": true })
router.push({ pathname: '/product/[id]', params: { id: '123' } });

// Read params
const { id } = useLocalSearchParams<{ id: string }>();
```

**Rules:**
- Enable `typedRoutes` in `app.json` for compile-time route validation.
- Use `router.replace` for auth flows to prevent back-navigation to the login screen.
- Use `(groups)` (folder name in parentheses) to group routes under a shared layout without adding a path segment.
- Use `+not-found.tsx` for 404 handling on web.

---

## React Navigation v7 (Code-based routing — alternative)

### Installation

```bash
npx expo install @react-navigation/native react-native-screens react-native-safe-area-context
npx expo install @react-navigation/native-stack  # for Stack
npx expo install @react-navigation/bottom-tabs   # for Tabs
```

### Static configuration (recommended)

```tsx
import { createStaticNavigation, createNativeStackNavigator } from '@react-navigation/native-stack';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';

const HomeTabs = createBottomTabNavigator({
  screens: {
    Home: HomeScreen,
    Profile: ProfileScreen,
  },
});

const RootStack = createNativeStackNavigator({
  screens: {
    HomeTabs,
    ProductDetail: { screen: ProductDetailScreen, linking: { path: 'product/:id' } },
  },
});

const Navigation = createStaticNavigation(RootStack);

export default function App() {
  return <Navigation />;
}
```

### Dynamic configuration

```tsx
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

type RootStackParamList = {
  Home: undefined;
  ProductDetail: { id: string };
};

const Stack = createNativeStackNavigator<RootStackParamList>();

export default function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Home">
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="ProductDetail" component={ProductDetailScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

**Rules:**
- Use `createNativeStackNavigator` (from `@react-navigation/native-stack`), NOT `createStackNavigator` — the native stack uses platform-native navigation controllers and is significantly faster.
- Always type `ParamList` for type-safe navigation and params.
- Disable `android:enableOnBackInvokedCallback` in `AndroidManifest.xml` until React Navigation fully supports Android predictive back.

---

## Deep Linking

### Expo Router (automatic)

```json
// app.json
{
  "expo": {
    "scheme": "myapp",
    "experiments": { "typedRoutes": true }
  }
}
```

### React Navigation v7 (manual)

```tsx
const linking = {
  prefixes: ['myapp://', 'https://myapp.com'],
  config: {
    screens: {
      Home: '',
      ProductDetail: 'product/:id',
    },
  },
};

<NavigationContainer linking={linking}>...</NavigationContainer>
```

---

## Safe Area

```tsx
import { SafeAreaView, SafeAreaProvider, useSafeAreaInsets } from 'react-native-safe-area-context';

// Wrap app root
<SafeAreaProvider>
  <App />
</SafeAreaProvider>

// In screen components
<SafeAreaView style={{ flex: 1 }}>
  <Content />
</SafeAreaView>

// Or use the hook for custom positioning
function Header() {
  const insets = useSafeAreaInsets();
  return <View style={{ paddingTop: insets.top }}>...</View>;
}
```

**Rules:**
- Always wrap the root app in `SafeAreaProvider`.
- Never hard-code status bar heights (`20`, `44`, `24`) — use `useSafeAreaInsets` or `SafeAreaView`.
- On Android, set `translucent={true}` on `StatusBar` for edge-to-edge layout; then use `insets.top` for offset.
