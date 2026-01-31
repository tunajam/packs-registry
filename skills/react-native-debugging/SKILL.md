# React Native Debugging

Comprehensive debugging techniques for React Native applications.

## When to Apply

- Debugging JS logic and state
- Profiling performance issues
- Investigating native crashes
- Network request debugging
- Layout debugging

## Dev Menu

Access the developer menu:
- **iOS Simulator**: Cmd + D
- **Android Emulator**: Cmd + M (Mac) or Ctrl + M (Windows)
- **Physical device**: Shake the device

## React Native Debugger Tools

### Expo Dev Tools
```bash
# Start with Expo
npx expo start

# Press j for JavaScript debugger
# Press Shift + m for dev menu
```

### Chrome DevTools (Hermes)
For Hermes-enabled apps (default in new RN):
```bash
# Open Chrome and navigate to
chrome://inspect

# Or use direct connection
# Metro will show: "Debugger connected" when ready
```

### Flipper (Deprecated in RN 0.73+)
For older projects:
```bash
# Install Flipper desktop app
brew install --cask flipper

# Ensure flipper is in Podfile (iOS)
use_flipper!({ 'Flipper' => '0.163.0' })
```

### React DevTools
```bash
# Install globally
npm install -g react-devtools

# Run
react-devtools

# Or use standalone
npx react-devtools
```

## Console Logging

### Structured Logging
```tsx
// Group related logs
console.group('User Authentication');
console.log('Attempting login...');
console.log('User:', username);
console.groupEnd();

// Tables for data
console.table(users);

// Timing
console.time('API Call');
await fetchData();
console.timeEnd('API Call'); // API Call: 234ms

// Conditional logging
console.assert(user != null, 'User should not be null');
```

### Remote Debugging
```tsx
// For production debugging
import { LogBox } from 'react-native';

// Ignore specific warnings
LogBox.ignoreLogs(['Warning: ...']);

// Ignore all warnings (not recommended)
LogBox.ignoreAllLogs();
```

## Network Debugging

### React Native Debugger
Built-in network inspector shows all requests.

### Using Proxyman/Charles
```tsx
// iOS: Configure in device settings
// Android: Add network security config

// android/app/src/main/res/xml/network_security_config.xml
<network-security-config>
  <debug-overrides>
    <trust-anchors>
      <certificates src="user" />
    </trust-anchors>
  </debug-overrides>
</network-security-config>

// android/app/src/main/AndroidManifest.xml
<application
  android:networkSecurityConfig="@xml/network_security_config"
  ...>
```

### Custom Network Logger
```tsx
// Intercept fetch
const originalFetch = global.fetch;
global.fetch = async (...args) => {
  console.log('Fetch request:', args[0]);
  const response = await originalFetch(...args);
  console.log('Fetch response:', response.status);
  return response;
};
```

## Performance Profiling

### React DevTools Profiler
1. Open React DevTools
2. Click "Profiler" tab
3. Click "Record"
4. Interact with app
5. Click "Stop" to analyze

### Systrace (Android)
```bash
# Start systrace
npx react-native profile-hermes

# Or use Android Studio Profiler
```

### Instruments (iOS)
```bash
# Profile in Xcode
Product > Profile > Time Profiler
```

### Performance Monitor
```tsx
// In-app performance overlay
// Enable via Dev Menu > Show Perf Monitor
```

### Custom Performance Marks
```tsx
import { PerformanceObserver, performance } from 'perf_hooks';

// Mark start
performance.mark('component-mount-start');

// ... component mounts ...

// Mark end
performance.mark('component-mount-end');

// Measure
performance.measure(
  'Component Mount',
  'component-mount-start',
  'component-mount-end'
);
```

## Layout Debugging

### Inspector
Enable via Dev Menu > Toggle Inspector

### Layout Borders
```tsx
// Visualize all layouts
const debugStyle = __DEV__ ? { borderWidth: 1, borderColor: 'red' } : {};

<View style={[styles.container, debugStyle]}>
```

### React Native Inspector
```tsx
// Built into DevTools
// Shows component hierarchy and props
```

## Native Debugging

### iOS (Xcode)
```bash
# Open in Xcode
open ios/MyApp.xcworkspace

# Set breakpoints in native code
# Use Console for native logs
# Use Debug Navigator for memory/CPU
```

### Android (Android Studio)
```bash
# Open android folder in Android Studio
# Use Logcat for logs
adb logcat *:S ReactNative:V ReactNativeJS:V

# Attach debugger for native breakpoints
```

### Common Native Crashes

**iOS:**
```bash
# Check crash logs
~/Library/Logs/DiagnosticReports/

# Symbolicate crash
xcrun atos -o MyApp.app.dSYM/Contents/Resources/DWARF/MyApp -arch arm64 -l 0x10000 0x123456
```

**Android:**
```bash
# Check logcat
adb logcat | grep -E "FATAL|AndroidRuntime"

# Get stack trace
adb bugreport > bugreport.zip
```

## Error Boundaries

```tsx
import React, { Component, ErrorInfo, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
  error?: Error;
}

class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    console.error('Error caught:', error);
    console.error('Component stack:', info.componentStack);
    
    // Report to error tracking service
    // Sentry.captureException(error);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || <ErrorScreen error={this.state.error} />;
    }
    return this.props.children;
  }
}
```

## Source Maps

### Enable Source Maps
```json
// metro.config.js
module.exports = {
  transformer: {
    minifierConfig: {
      sourceMap: {
        includeSources: true,
      },
    },
  },
};
```

### Upload to Error Tracking
```bash
# Sentry example
sentry-cli react-native gradle --bundle ./index.android.bundle --sourcemap ./index.android.bundle.map
```

## Best Practices

1. **Use TypeScript** - Catch errors at compile time
2. **Add Error Boundaries** - Graceful error handling
3. **Log strategically** - Remove console.log in production
4. **Profile regularly** - Don't wait for problems
5. **Test on devices** - Simulators behave differently
6. **Use source maps** - Readable stack traces in production
7. **Monitor crashes** - Use Sentry, Crashlytics, or similar
8. **Reproduce issues** - Create minimal test cases
