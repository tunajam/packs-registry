# React Native Gesture Handler

Native-driven gesture handling for smooth touch interactions.

## When to Apply

- Building swipeable cards
- Pull-to-refresh implementations
- Drag and drop interfaces
- Pinch-to-zoom
- Complex multi-touch gestures

## Installation

```bash
# Expo
npx expo install react-native-gesture-handler

# React Native CLI
npm install react-native-gesture-handler
cd ios && pod install
```

### Setup
```tsx
// App.tsx - Wrap root component
import { GestureHandlerRootView } from 'react-native-gesture-handler';

export default function App() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <AppContent />
    </GestureHandlerRootView>
  );
}
```

## Gesture API (v2)

### Basic Structure
```tsx
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

function MyComponent() {
  const gesture = Gesture.Tap()
    .onStart(() => console.log('Tap started'))
    .onEnd(() => console.log('Tap ended'));

  return (
    <GestureDetector gesture={gesture}>
      <View style={styles.box} />
    </GestureDetector>
  );
}
```

## Gesture Types

### Tap Gesture
```tsx
const tap = Gesture.Tap()
  .numberOfTaps(2) // Double tap
  .maxDuration(300)
  .onStart((event) => {
    'worklet';
    console.log(`Tapped at: ${event.x}, ${event.y}`);
  })
  .onEnd((event, success) => {
    'worklet';
    if (success) {
      runOnJS(handleTap)();
    }
  });
```

### Pan Gesture
```tsx
const translateX = useSharedValue(0);
const translateY = useSharedValue(0);

const pan = Gesture.Pan()
  .minDistance(10)
  .onStart((event) => {
    'worklet';
    // Save starting position
  })
  .onUpdate((event) => {
    'worklet';
    translateX.value = event.translationX;
    translateY.value = event.translationY;
  })
  .onEnd((event) => {
    'worklet';
    // event.velocityX, event.velocityY available
    translateX.value = withSpring(0);
    translateY.value = withSpring(0);
  });
```

### Pinch Gesture
```tsx
const scale = useSharedValue(1);
const savedScale = useSharedValue(1);

const pinch = Gesture.Pinch()
  .onStart(() => {
    'worklet';
    savedScale.value = scale.value;
  })
  .onUpdate((event) => {
    'worklet';
    scale.value = savedScale.value * event.scale;
  })
  .onEnd(() => {
    'worklet';
    scale.value = withSpring(1);
  });
```

### Rotation Gesture
```tsx
const rotation = useSharedValue(0);
const savedRotation = useSharedValue(0);

const rotate = Gesture.Rotation()
  .onStart(() => {
    'worklet';
    savedRotation.value = rotation.value;
  })
  .onUpdate((event) => {
    'worklet';
    rotation.value = savedRotation.value + event.rotation;
  });
```

### Long Press
```tsx
const longPress = Gesture.LongPress()
  .minDuration(800) // 800ms hold
  .onStart(() => {
    'worklet';
    runOnJS(hapticFeedback)();
    runOnJS(showContextMenu)();
  });
```

### Fling Gesture
```tsx
const fling = Gesture.Fling()
  .direction(Directions.RIGHT | Directions.LEFT)
  .onStart((event) => {
    'worklet';
    if (event.x > 0) {
      // Flinged right
    } else {
      // Flinged left
    }
  });
```

## Combining Gestures

### Simultaneous
```tsx
// Both gestures can be active at same time
const composed = Gesture.Simultaneous(pinch, rotate);

// Usage
<GestureDetector gesture={composed}>
  <Animated.View style={animatedStyle} />
</GestureDetector>
```

### Exclusive
```tsx
// Only one gesture active at a time (first wins)
const composed = Gesture.Exclusive(longPress, tap);
```

### Race
```tsx
// First to activate wins
const composed = Gesture.Race(pan, tap);
```

### Sequence
```tsx
// Must happen in order
const composed = Gesture.Sequence(tap, longPress);
```

## Gesture Configuration

### Hit Slop
```tsx
const gesture = Gesture.Tap()
  .hitSlop({ left: 20, right: 20, top: 10, bottom: 10 });
```

### Activation Criteria
```tsx
const pan = Gesture.Pan()
  .minDistance(10) // Min pixels to activate
  .minPointers(1)
  .maxPointers(2)
  .activateAfterLongPress(500)
  .activeOffsetX([-20, 20]) // Activate when moving 20px horizontally
  .failOffsetY([-5, 5]); // Fail if moving 5px vertically
```

### Enable/Disable
```tsx
const [enabled, setEnabled] = useState(true);

const gesture = Gesture.Pan()
  .enabled(enabled);
```

## Common Patterns

### Swipe to Delete
```tsx
function SwipeableRow({ onDelete, children }) {
  const translateX = useSharedValue(0);
  const threshold = -100;

  const pan = Gesture.Pan()
    .activeOffsetX([-10, 10])
    .onUpdate((event) => {
      'worklet';
      translateX.value = Math.max(event.translationX, -150);
    })
    .onEnd((event) => {
      'worklet';
      if (translateX.value < threshold) {
        translateX.value = withTiming(-150);
        runOnJS(onDelete)();
      } else {
        translateX.value = withSpring(0);
      }
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }],
  }));

  return (
    <View style={styles.container}>
      <View style={styles.deleteButton}>
        <Text>Delete</Text>
      </View>
      <GestureDetector gesture={pan}>
        <Animated.View style={[styles.row, animatedStyle]}>
          {children}
        </Animated.View>
      </GestureDetector>
    </View>
  );
}
```

### Pull to Refresh
```tsx
function PullToRefresh({ onRefresh, children }) {
  const translateY = useSharedValue(0);
  const isRefreshing = useSharedValue(false);
  const refreshThreshold = 100;

  const pan = Gesture.Pan()
    .onUpdate((event) => {
      'worklet';
      if (!isRefreshing.value && event.translationY > 0) {
        translateY.value = event.translationY * 0.5; // Resistance
      }
    })
    .onEnd(() => {
      'worklet';
      if (translateY.value > refreshThreshold) {
        isRefreshing.value = true;
        translateY.value = withTiming(refreshThreshold);
        runOnJS(onRefresh)(() => {
          isRefreshing.value = false;
          translateY.value = withSpring(0);
        });
      } else {
        translateY.value = withSpring(0);
      }
    });

  return (
    <GestureDetector gesture={pan}>
      <Animated.View style={useAnimatedStyle(() => ({
        transform: [{ translateY: translateY.value }],
      }))}>
        {children}
      </Animated.View>
    </GestureDetector>
  );
}
```

### Draggable Card
```tsx
function DraggableCard() {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const context = useSharedValue({ x: 0, y: 0 });
  const isPressed = useSharedValue(false);

  const pan = Gesture.Pan()
    .onStart(() => {
      'worklet';
      context.value = { x: translateX.value, y: translateY.value };
      isPressed.value = true;
    })
    .onUpdate((event) => {
      'worklet';
      translateX.value = context.value.x + event.translationX;
      translateY.value = context.value.y + event.translationY;
    })
    .onEnd(() => {
      'worklet';
      isPressed.value = false;
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { scale: withSpring(isPressed.value ? 1.1 : 1) },
    ],
    shadowOpacity: withSpring(isPressed.value ? 0.3 : 0.1),
  }));

  return (
    <GestureDetector gesture={pan}>
      <Animated.View style={[styles.card, animatedStyle]} />
    </GestureDetector>
  );
}
```

## Best Practices

1. **Use worklets** - All gesture callbacks should have `'worklet'` directive
2. **Combine gestures wisely** - Use Simultaneous for multi-touch, Exclusive for alternatives
3. **Add resistance** - For pulls/drags that have limits
4. **Use spring for endings** - More natural than timing
5. **Handle interruptions** - Gestures can be cancelled
6. **Test on device** - Simulators don't fully replicate touch
7. **Profile performance** - Ensure 60fps
