# Hermes Engine for React Native

Optimize React Native apps with Meta's Hermes JavaScript engine.

## When to Apply

- Reducing app startup time
- Lowering memory usage
- Improving TTI (Time to Interactive)
- Debugging with Hermes
- Understanding bytecode compilation

## What is Hermes?

Hermes is a JavaScript engine optimized for React Native:
- **Ahead-of-time (AOT) compilation** - Compiles JS to bytecode at build time
- **Smaller bundle size** - Bytecode is more compact
- **Faster startup** - No JIT compilation needed at runtime
- **Lower memory** - Optimized garbage collection

## Enabling Hermes

### React Native 0.70+ (Default)
Hermes is enabled by default. To verify:

```tsx
const isHermes = () => !!global.HermesInternal;
console.log('Using Hermes:', isHermes());
```

### Expo
```json
// app.json
{
  "expo": {
    "jsEngine": "hermes"
  }
}
```

### React Native CLI (if disabled)

**iOS (Podfile):**
```ruby
use_react_native!(
  :hermes_enabled => true
)
```

**Android (gradle.properties):**
```properties
hermesEnabled=true
```

## Performance Benefits

### Typical Improvements
| Metric | Without Hermes | With Hermes |
|--------|----------------|-------------|
| App Size | 100% | ~85% |
| TTI | 100% | ~60% |
| Memory | 100% | ~70% |

### Measuring Impact
```tsx
// Log app startup time
const startTime = global.nativePerformanceNow?.() || Date.now();

function App() {
  useEffect(() => {
    const endTime = global.nativePerformanceNow?.() || Date.now();
    console.log(`Startup time: ${endTime - startTime}ms`);
  }, []);
  
  return <MainApp />;
}
```

## Debugging with Hermes

### Chrome DevTools
```bash
# Open Chrome and navigate to
chrome://inspect

# Or use React Native DevTools
npx react-devtools
```

### Hermes Debugger
```bash
# Start Metro with debugging
npx react-native start

# Press 'j' in Metro to open debugger
```

### Source Maps
Hermes generates source maps for debugging:

```js
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

## Hermes-Specific Features

### Intl Support
Hermes has limited Intl support. Add polyfills if needed:

```bash
npm install intl-pluralrules @formatjs/intl-getcanonicallocales \
  @formatjs/intl-locale @formatjs/intl-numberformat \
  @formatjs/intl-datetimeformat
```

```tsx
// index.js (before App import)
import 'intl-pluralrules';
import '@formatjs/intl-getcanonicallocales/polyfill';
import '@formatjs/intl-locale/polyfill';
import '@formatjs/intl-numberformat/polyfill';
import '@formatjs/intl-numberformat/locale-data/en';
import '@formatjs/intl-datetimeformat/polyfill';
import '@formatjs/intl-datetimeformat/locale-data/en';
```

### Checking Hermes Version
```tsx
if (global.HermesInternal) {
  const hermesVersion = global.HermesInternal.getRuntimeProperties()['OSS Release Version'];
  console.log('Hermes version:', hermesVersion);
}
```

## Profiling

### CPU Profiling
```bash
# Generate CPU profile
npx react-native profile-hermes

# This creates a .cpuprofile file
# Open in Chrome DevTools > Performance tab
```

### Memory Profiling
```tsx
// Get heap stats
if (global.HermesInternal) {
  const heapInfo = global.HermesInternal.getInstrumentedStats();
  console.log('Heap size:', heapInfo.js_heapSize);
  console.log('Heap used:', heapInfo.js_heapUsed);
}
```

### Sampling Profiler
```tsx
import { unstable_enableSamplingProfiler } from 'react-native';

// Enable profiler
unstable_enableSamplingProfiler?.();

// Disable and get results
const profile = unstable_getSamplingProfileData?.();
```

## Bytecode Compilation

### How It Works
1. Metro bundles JavaScript
2. Hermes compiler (`hermesc`) compiles to bytecode
3. Bytecode (.hbc) is included in app
4. App loads bytecode directly (no parsing)

### Build-Time Compilation
```gradle
// android/app/build.gradle
project.ext.react = [
    enableHermes: true,
    hermesFlagsRelease: ["-O", "-output-source-map"],
]
```

### Precompiled Bundles
```bash
# Compile JS to Hermes bytecode
npx react-native bundle \
  --platform android \
  --dev false \
  --entry-file index.js \
  --bundle-output android/app/src/main/assets/index.android.bundle

# Compile bytecode
node_modules/react-native/sdks/hermesc/osx-bin/hermesc \
  -emit-binary \
  -out android/app/src/main/assets/index.android.bundle.hbc \
  android/app/src/main/assets/index.android.bundle
```

## Optimization Tips

### 1. Minimize Bundle Size
```js
// metro.config.js
module.exports = {
  transformer: {
    minifierPath: 'metro-minify-terser',
    minifierConfig: {
      mangle: true,
      compress: {
        drop_console: true,
        drop_debugger: true,
      },
    },
  },
};
```

### 2. Code Splitting (Dynamic Imports)
```tsx
// Lazy load heavy components
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));

function Screen() {
  return (
    <Suspense fallback={<Loading />}>
      <HeavyComponent />
    </Suspense>
  );
}
```

### 3. Avoid Large JSON Imports
```tsx
// ❌ Bad - entire file parsed
import data from './large-data.json';

// ✅ Better - fetch at runtime
const data = await fetch('./large-data.json').then(r => r.json());
```

### 4. Use RAM Bundles (Advanced)
```bash
# iOS
npx react-native bundle \
  --platform ios \
  --dev false \
  --entry-file index.js \
  --bundle-output ios/main.jsbundle \
  --indexed-ram-bundle
```

## Compatibility Notes

### Unsupported Features
- `with` statement (strict mode)
- Legacy decorators (use Babel)
- Some Proxy traps (limited support)
- Full Intl API (needs polyfills)

### Polyfills to Consider
```tsx
// For older Hermes versions
import 'react-native-url-polyfill/auto';
import 'text-encoding-polyfill';
```

## Troubleshooting

### Build Errors
```bash
# Clean and rebuild
cd android && ./gradlew clean
cd ios && pod install --repo-update
npx react-native start --reset-cache
```

### Runtime Errors
```tsx
// Check if Hermes-related
if (!global.HermesInternal) {
  console.warn('Not running on Hermes');
}
```

### Performance Issues
1. Profile with Chrome DevTools
2. Check for large bundle size
3. Verify release build (not debug)
4. Look for synchronous operations

## Best Practices

1. **Always use Hermes** - Default for good reason
2. **Profile in release mode** - Debug mode is slow
3. **Add Intl polyfills** - If using internationalization
4. **Monitor bundle size** - Smaller = faster startup
5. **Test on low-end devices** - Where benefits are most visible
6. **Use source maps** - For debugging production issues
7. **Keep dependencies updated** - Hermes improves constantly
