# React Native — Performance

Version target: React Native 0.76+ / Reanimated 3.x / React Navigation 7

---

## The Two Threads

| Thread | Role | What blocks it |
|--------|------|----------------|
| **JS thread** | Business logic, React reconciliation, state updates, API calls | Heavy computation, large re-renders, unthrottled event handlers |
| **UI thread (main thread)** | Native view drawing, touch handling, native animations | Expensive layout passes, many overlapping transparent views |

**Target:** 60 fps = 16.67 ms per frame. Drop below this on either thread and users see jank.

---

## Animation

### Animated API — always use `useNativeDriver: true`

```tsx
import { Animated, useRef } from 'react-native';

// ✅ Runs on the UI thread — no JS involvement per frame
const opacity = useRef(new Animated.Value(0)).current;

Animated.timing(opacity, {
  toValue: 1,
  duration: 300,
  useNativeDriver: true, // ← required for transform and opacity
}).start();

<Animated.View style={{ opacity }} />
```

**Rules:**
- `useNativeDriver: true` only works for `transform` and `opacity` — NOT for `width`, `height`, `flex`, or `backgroundColor`.
- For layout animations, use `LayoutAnimation` (fire-and-forget) or Reanimated 3.
- For interruptible, physics-based, or gesture-driven animations, use **Reanimated 3**.

### Reanimated 3 (recommended for complex animations)

```tsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  runOnJS,
} from 'react-native-reanimated';

function Card() {
  const scale = useSharedValue(1);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  return (
    <Animated.View style={[styles.card, animatedStyle]}>
      <Pressable
        onPressIn={() => { scale.value = withSpring(0.95); }}
        onPressOut={() => { scale.value = withSpring(1); }}
      >
        <Text>Press me</Text>
      </Pressable>
    </Animated.View>
  );
}
```

**Rules:**
- Shared values live on the UI thread — reads/writes in worklets are synchronous.
- `useAnimatedStyle` worklets must be pure (no side effects, no JS thread calls).
- To call JS thread functions from a worklet, use `runOnJS(myFn)(arg)`.

### LayoutAnimation

```tsx
import { LayoutAnimation, Platform, UIManager } from 'react-native';

// Enable on Android
if (Platform.OS === 'android') {
  UIManager.setLayoutAnimationEnabledExperimental?.(true);
}

LayoutAnimation.configureNext(LayoutAnimation.Presets.easeInEaseOut);
setExpanded(!expanded);
```

**Rules:**
- `LayoutAnimation` is not interruptible — use Reanimated for complex choreography.
- Must enable experimentally on Android via `UIManager.setLayoutAnimationEnabledExperimental`.

---

## FlatList Performance

### Critical props

| Prop | Default | Recommendation |
|------|---------|---------------|
| `initialNumToRender` | 10 | Set to fill one screen height |
| `maxToRenderPerBatch` | 10 | Increase for smoother scroll, decrease for responsiveness |
| `windowSize` | 21 | Decrease (e.g. 5) for memory-constrained devices |
| `updateCellsBatchingPeriod` | 50ms | Balance with maxToRenderPerBatch |
| `removeClippedSubviews` | true (Android) | Keep true; beware iOS visual bugs with complex transforms |
| `getItemLayout` | — | **Required** for fixed-height items; enables `scrollToIndex` |

### Anti-patterns

```tsx
// ❌ Anonymous renderItem — new function reference every render
<FlatList data={items} renderItem={({ item }) => <Item data={item} />} />

// ✅ Stable reference with useCallback
const renderItem = useCallback(({ item }: ListRenderItemInfo<MyItem>) => (
  <Item data={item} />
), []);

// ❌ Missing keyExtractor — falls back to index, breaks animations on reorder
<FlatList data={items} renderItem={renderItem} />

// ✅ Stable, unique key from data
const keyExtractor = useCallback((item: MyItem) => item.id, []);

// ✅ Memoize the list item component
const ListItem = memo(({ data }: { data: MyItem }) => (
  <View style={styles.item}>
    <Text>{data.title}</Text>
  </View>
));
```

### For massive lists (>500 items): FlashList

```tsx
import { FlashList } from '@shopify/flash-list';

<FlashList
  data={items}
  renderItem={renderItem}
  estimatedItemSize={80} // ← average item height; required
  keyExtractor={keyExtractor}
/>
```

---

## JS Thread Performance

### InteractionManager — defer work until animations complete

```tsx
import { InteractionManager } from 'react-native';

useEffect(() => {
  const task = InteractionManager.runAfterInteractions(() => {
    loadExpensiveData();
  });
  return () => task.cancel();
}, []);
```

### Wrap expensive `onPress` in requestAnimationFrame

```tsx
function handlePress() {
  requestAnimationFrame(() => {
    performExpensiveOperation();
  });
}
```

---

## Re-render Prevention

```tsx
// ✅ Memoize child components
const ExpensiveChild = memo(({ label }: { label: string }) => (
  <Text>{label}</Text>
));

// ✅ Stable callback references
const handleChange = useCallback((text: string) => {
  dispatch({ type: 'SET_TEXT', payload: text });
}, [dispatch]);
```

**Rules:**
- Only memoize when you have measured a re-render problem.
- With New Architecture + concurrent mode, multiple `setState` calls in async code batch automatically.

---

## Remove console.log in Production

```js
// babel.config.js
module.exports = {
  plugins: [
    ['transform-remove-console', { exclude: ['error', 'warn'] }],
  ],
};
```

---

## Performance Profiling Tools

| Tool | How to use | Finds |
|------|-----------|-------|
| **Perf Monitor** (Dev Menu) | Shake device → Show Perf Monitor | JS/UI frame rates live |
| **Hermes Sampling Profiler** | Dev Menu → Start/Stop Sampling Profiler | JS CPU bottlenecks |
| **React DevTools Profiler** | Via Flipper or standalone | Component render times |
| **Xcode Instruments** (iOS) | Instruments > Time Profiler | Native CPU/GPU profiling |
| **Android Profiler** | Android Studio | Native thread timings |

**Always profile release builds** — development mode adds significant overhead.
