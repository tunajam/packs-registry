# React Native Reanimated

High-performance animations running on the UI thread.

## When to Apply

- Building smooth, 60fps animations
- Gesture-driven interactions
- Complex spring/timing animations
- Worklet-based animations

## Installation

```bash
# Expo
npx expo install react-native-reanimated

# React Native CLI
npm install react-native-reanimated
cd ios && pod install
```

### Babel Config
```js
// babel.config.js
module.exports = {
  presets: ['module:@react-native/babel-preset'],
  plugins: ['react-native-reanimated/plugin'], // Must be last
};
```

## Core Concepts

### Shared Values
```tsx
import Animated, { useSharedValue, useAnimatedStyle } from 'react-native-reanimated';

function MyComponent() {
  // Shared value - can be read/written from UI thread
  const offset = useSharedValue(0);
  
  // Animated style based on shared value
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: offset.value }],
  }));
  
  return <Animated.View style={[styles.box, animatedStyle]} />;
}
```

### Worklets
```tsx
import { runOnJS, runOnUI } from 'react-native-reanimated';

// Worklet - runs on UI thread
const myWorklet = () => {
  'worklet';
  // This runs on UI thread
  return someValue.value * 2;
};

// Call JS from worklet
function updateState(value: number) {
  setState(value);
}

const workletWithJS = () => {
  'worklet';
  runOnJS(updateState)(42);
};
```

## Animation Types

### withTiming
```tsx
import { withTiming, Easing } from 'react-native-reanimated';

// Simple timing
offset.value = withTiming(100);

// With duration and easing
offset.value = withTiming(100, {
  duration: 500,
  easing: Easing.bezier(0.25, 0.1, 0.25, 1),
});

// With callback
offset.value = withTiming(100, {}, (finished) => {
  if (finished) {
    runOnJS(onAnimationComplete)();
  }
});
```

### withSpring
```tsx
import { withSpring } from 'react-native-reanimated';

// Default spring
offset.value = withSpring(100);

// Custom spring physics
offset.value = withSpring(100, {
  damping: 20,
  stiffness: 90,
  mass: 1,
  overshootClamping: false,
  restDisplacementThreshold: 0.01,
  restSpeedThreshold: 0.01,
});

// Using velocity
offset.value = withSpring(100, {
  velocity: currentVelocity,
});
```

### withDecay
```tsx
import { withDecay } from 'react-native-reanimated';

// Natural deceleration (like scrolling)
offset.value = withDecay({
  velocity: velocityFromGesture,
  deceleration: 0.998,
});

// With clamp
offset.value = withDecay({
  velocity: velocity,
  clamp: [0, 500], // Min and max bounds
});
```

### Combining Animations

```tsx
import { withSequence, withRepeat, withDelay } from 'react-native-reanimated';

// Sequence
offset.value = withSequence(
  withTiming(100),
  withTiming(0),
  withTiming(100)
);

// Repeat
offset.value = withRepeat(
  withTiming(100, { duration: 1000 }),
  -1, // Infinite
  true // Reverse
);

// Delay
offset.value = withDelay(500, withTiming(100));
```

## Animated Components

### Built-in Components
```tsx
import Animated from 'react-native-reanimated';

<Animated.View style={animatedStyle} />
<Animated.Text style={animatedStyle} />
<Animated.Image style={animatedStyle} />
<Animated.ScrollView />
<Animated.FlatList />
```

### Create Animated Component
```tsx
import Animated from 'react-native-reanimated';
import { TouchableOpacity } from 'react-native';

const AnimatedTouchable = Animated.createAnimatedComponent(TouchableOpacity);
```

## Hooks

### useDerivedValue
```tsx
import { useDerivedValue } from 'react-native-reanimated';

const x = useSharedValue(0);
const y = useSharedValue(0);

// Compute derived value
const diagonal = useDerivedValue(() => {
  return Math.sqrt(x.value ** 2 + y.value ** 2);
});
```

### useAnimatedReaction
```tsx
import { useAnimatedReaction } from 'react-native-reanimated';

useAnimatedReaction(
  () => offset.value, // Prepare - what to track
  (currentValue, previousValue) => { // React - what to do
    if (currentValue > 100 && previousValue <= 100) {
      runOnJS(hapticFeedback)();
    }
  }
);
```

### useAnimatedScrollHandler
```tsx
import { useAnimatedScrollHandler, useSharedValue } from 'react-native-reanimated';

const scrollY = useSharedValue(0);

const scrollHandler = useAnimatedScrollHandler({
  onScroll: (event) => {
    scrollY.value = event.contentOffset.y;
  },
  onBeginDrag: () => {
    console.log('Started scrolling');
  },
  onEndDrag: () => {
    console.log('Stopped scrolling');
  },
});

<Animated.ScrollView onScroll={scrollHandler} scrollEventThrottle={16}>
  {/* content */}
</Animated.ScrollView>
```

## Layout Animations

### Entering/Exiting
```tsx
import Animated, { FadeIn, FadeOut, SlideInRight } from 'react-native-reanimated';

<Animated.View
  entering={FadeIn.duration(500)}
  exiting={FadeOut.duration(300)}
>
  <Text>Animated content</Text>
</Animated.View>

// Custom entering
<Animated.View
  entering={SlideInRight.springify().damping(15)}
>
  <Text>Slides in</Text>
</Animated.View>
```

### Layout Transition
```tsx
import Animated, { LinearTransition } from 'react-native-reanimated';

<Animated.View layout={LinearTransition.springify()}>
  {items.map(item => (
    <Animated.View
      key={item.id}
      layout={LinearTransition}
      entering={FadeIn}
      exiting={FadeOut}
    >
      <Text>{item.name}</Text>
    </Animated.View>
  ))}
</Animated.View>
```

## Gesture Integration

```tsx
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, { useSharedValue, useAnimatedStyle, withSpring } from 'react-native-reanimated';

function DraggableBox() {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const context = useSharedValue({ x: 0, y: 0 });

  const gesture = Gesture.Pan()
    .onStart(() => {
      context.value = { x: translateX.value, y: translateY.value };
    })
    .onUpdate((event) => {
      translateX.value = context.value.x + event.translationX;
      translateY.value = context.value.y + event.translationY;
    })
    .onEnd(() => {
      translateX.value = withSpring(0);
      translateY.value = withSpring(0);
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
    ],
  }));

  return (
    <GestureDetector gesture={gesture}>
      <Animated.View style={[styles.box, animatedStyle]} />
    </GestureDetector>
  );
}
```

## Interpolation

```tsx
import { interpolate, Extrapolation } from 'react-native-reanimated';

const animatedStyle = useAnimatedStyle(() => {
  const opacity = interpolate(
    scrollY.value,
    [0, 100, 200],
    [1, 0.5, 0],
    Extrapolation.CLAMP
  );
  
  const scale = interpolate(
    scrollY.value,
    [0, 100],
    [1, 1.5]
  );
  
  return { opacity, transform: [{ scale }] };
});
```

### Color Interpolation
```tsx
import { interpolateColor } from 'react-native-reanimated';

const animatedStyle = useAnimatedStyle(() => {
  const backgroundColor = interpolateColor(
    progress.value,
    [0, 1],
    ['#FF0000', '#00FF00']
  );
  
  return { backgroundColor };
});
```

## Best Practices

1. **Keep worklets small** - Complex logic should stay in JS
2. **Use derived values** - Avoid redundant calculations
3. **Clamp values** - Prevent out-of-bounds animations
4. **Profile performance** - Check for JS thread drops
5. **Avoid re-creating styles** - Use useMemo for static parts
6. **Use spring for natural feel** - Timing for precise control
7. **Handle interruptions** - Animations can be interrupted
