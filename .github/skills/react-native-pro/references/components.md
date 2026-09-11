# React Native — Core Components & Styling

Version target: React Native 0.76+ / Expo SDK 52+

---

## Core Components Quick Reference

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `View` | Layout container (like `div`) | `style`, `accessible`, `testID` |
| `Text` | Text rendering | `style`, `numberOfLines`, `onPress`, `selectable` |
| `Image` | Raster images | `source`, `resizeMode`, `onLoad`, `onError` |
| `TextInput` | Text entry | `value`, `onChangeText`, `placeholder`, `keyboardType`, `returnKeyType` |
| `Pressable` | Touch target | `onPress`, `onLongPress`, `android_ripple`, `style` (function form) |
| `ScrollView` | Scrollable container for few items | `horizontal`, `contentContainerStyle`, `keyboardShouldPersistTaps` |
| `FlatList` | Virtualized list for large datasets | `data`, `renderItem`, `keyExtractor`, `getItemLayout` |
| `SectionList` | Grouped/sectioned list | `sections`, `renderItem`, `renderSectionHeader`, `keyExtractor` |
| `Modal` | Overlay content | `visible`, `transparent`, `animationType`, `onRequestClose` |
| `ActivityIndicator` | Loading spinner | `size`, `color`, `animating` |
| `Switch` | Boolean toggle | `value`, `onValueChange`, `trackColor`, `thumbColor` |

---

## `View`

```tsx
import { View, StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#fff' },
});

// ✅ Correct: use StyleSheet.create, set accessibility props
<View style={styles.container} accessible accessibilityLabel="Main content">
  {children}
</View>
```

**Rules:**
- Prefer `StyleSheet.create` over inline objects — inline objects are new references every render, causing unnecessary re-renders of memoized children.
- Never nest a `View` inside `Text` — it works on iOS but throws on Android.
- `overflow: 'hidden'` is required to clip child views on Android; it is automatic on iOS.

---

## `Text`

```tsx
// ✅ Nested Text for inline styling
<Text style={styles.body}>
  Hello, <Text style={styles.bold}>World</Text>!
</Text>

// ❌ Anti-pattern: non-Text children inside Text
<Text>
  <View /> {/* throws on Android */}
</Text>
```

**Rules:**
- All string literals that are direct children of a `View` must be wrapped in `<Text>`.
- Use `numberOfLines` + `ellipsizeMode` for truncation, not CSS `text-overflow`.

---

## `Image`

```tsx
import { Image } from 'react-native';

// ✅ Network image — always provide width/height or flex
<Image
  source={{ uri: 'https://example.com/photo.jpg' }}
  style={{ width: 200, height: 150 }}
  resizeMode="cover"
  onError={({ nativeEvent }) => console.warn(nativeEvent.error)}
/>

// ✅ Static asset
<Image source={require('./assets/logo.png')} style={styles.logo} />
```

**Rules:**
- Network `Image` components **must** have explicit dimensions; they render as 0×0 without them.
- Use `expo-image` or `@d11/react-native-fast-image` for caching, priority loading, and progressive rendering.
- Avoid animating `width`/`height` on `Image` (causes re-crop on iOS); use `transform: [{ scale }]` instead.

---

## `Pressable`

```tsx
// ✅ style as a function responds to press state
<Pressable
  onPress={handlePress}
  style={({ pressed }) => [styles.button, pressed && styles.pressed]}
  android_ripple={{ color: '#ccc', borderless: false }}
  accessibilityRole="button"
  accessibilityLabel="Submit form"
>
  <Text style={styles.label}>Submit</Text>
</Pressable>
```

**Rules:**
- Prefer `Pressable` over deprecated `TouchableOpacity`, `TouchableHighlight`, `TouchableNativeFeedback`.
- Always set `accessibilityRole` and `accessibilityLabel` on interactive elements.
- Wrap expensive work inside `onPress` in `requestAnimationFrame` if it causes dropped frames.

---

## `FlatList`

```tsx
// ✅ Correct pattern
const renderItem = useCallback(({ item }: { item: Todo }) => (
  <TodoItem todo={item} />
), []);

const keyExtractor = useCallback((item: Todo) => item.id.toString(), []);

<FlatList
  data={todos}
  renderItem={renderItem}
  keyExtractor={keyExtractor}
  getItemLayout={(_, index) => ({ length: ITEM_HEIGHT, offset: ITEM_HEIGHT * index, index })}
  initialNumToRender={10}
  maxToRenderPerBatch={10}
  windowSize={21}
  removeClippedSubviews={true}
/>
```

**Rules:**
- `renderItem` and `keyExtractor` **must** be stable references (declared outside render or wrapped in `useCallback`).
- `getItemLayout` is required for lists with fixed-height items — enables scroll-to-index without layout measurement.
- For lists >500 items, replace `FlatList` with FlashList or LegendList.
- Never use `ScrollView` for long lists — it renders all items at once.

---

## `ScrollView`

```tsx
<ScrollView
  contentContainerStyle={styles.contentContainer}
  keyboardShouldPersistTaps="handled"
  showsVerticalScrollIndicator={false}
>
  <FormField />
</ScrollView>
```

**Rules:**
- `keyboardShouldPersistTaps="handled"` is required for forms with `TextInput` inside `ScrollView`.
- Use `contentContainerStyle` (not `style`) to pad scroll content.

---

## StyleSheet API

```tsx
import { StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#ffffff',
    padding: 16,
  },
  title: {
    fontSize: 24,
    fontWeight: '700',
    color: '#111',
  },
});
```

**Rules:**
- Always use `StyleSheet.create` — validates styles in dev, freezes/IDs the object in production.
- Style property names are camelCase (`backgroundColor`, not `background-color`).
- `StyleSheet` does NOT support all CSS — no `transition`, `grid`, `:hover`, `::before/after`, `em/rem` units.
- Use `StyleSheet.absoluteFill` for `{ position: 'absolute', top: 0, left: 0, right: 0, bottom: 0 }`.

---

## Flexbox in React Native

| Property | Default | Notes |
|----------|---------|-------|
| `flexDirection` | `'column'` | RN default is column (opposite of CSS web) |
| `alignItems` | `'stretch'` | |
| `justifyContent` | `'flex-start'` | |
| `flex` | — | Shorthand for flexGrow + flexShrink + flexBasis |

**Rules:**
- Default flex direction is `column`, not `row` as in CSS web.
- `flex: 1` is more idiomatic than `width: '100%'` for filling parent space.
- `gap` is supported since RN 0.71.

---

## Accessibility

```tsx
<Pressable
  accessibilityRole="button"
  accessibilityLabel="Delete item"
  accessibilityHint="Removes this item from the list"
  accessibilityState={{ disabled: isLoading }}
  onPress={handleDelete}
>
  <Icon name="trash" />
</Pressable>
```

**Key props:**
- `accessible` — marks view as a single accessibility element
- `accessibilityRole` — `button`, `link`, `image`, `header`, `checkbox`, `tab`, etc.
- `accessibilityLabel` — read by VoiceOver/TalkBack instead of children
- `accessibilityHint` — describes what will happen when activated
- `accessibilityState` — `{ disabled, selected, checked, busy, expanded }`

**Rules:**
- Every `Pressable` needs `accessibilityRole` and `accessibilityLabel`.
- Images that convey information need `accessibilityLabel`; decorative images use `accessible={false}`.
- Test with VoiceOver (iOS) and TalkBack (Android) before shipping.
