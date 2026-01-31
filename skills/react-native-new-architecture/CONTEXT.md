# React Native New Architecture

Understanding and migrating to React Native's New Architecture.

## When to Apply

- Starting new React Native projects
- Migrating existing apps to New Architecture
- Writing native modules or Turbo Modules
- Understanding Fabric renderer
- Performance optimization

## Architecture Overview

### Old Architecture
```
JS Thread ←→ Bridge (JSON serialization) ←→ Native Thread
```
- All communication serialized as JSON
- Asynchronous message passing
- Performance bottleneck on heavy data transfer

### New Architecture
```
JS Thread ←→ JSI (direct binding) ←→ Native Thread
```
- Direct memory access via JSI (JavaScript Interface)
- Synchronous calls when needed
- No JSON serialization overhead

## Key Components

### JSI (JavaScript Interface)
- C++ layer enabling direct JS ↔ Native communication
- Allows JS to hold references to C++ objects
- Enables synchronous native calls

### Turbo Modules
- New native module system
- Lazy loading (modules load when first used)
- Type-safe with Codegen
- Synchronous method support

### Fabric
- New rendering system
- Concurrent rendering support
- Improved layout handling
- Direct manipulation of native views

### Codegen
- Generates native code from JS specs
- Ensures type safety across the bridge
- Eliminates runtime type errors

## Enabling New Architecture

### Expo (SDK 51+)
```json
// app.json
{
  "expo": {
    "newArchEnabled": true
  }
}
```

### React Native CLI (0.76+)
New Architecture is enabled by default in RN 0.76+.

For older versions:

**iOS (Podfile):**
```ruby
ENV['RCT_NEW_ARCH_ENABLED'] = '1'
```

**Android (gradle.properties):**
```properties
newArchEnabled=true
```

Then rebuild:
```bash
# iOS
cd ios && pod install && cd ..
npx react-native run-ios

# Android
npx react-native run-android
```

## Writing Turbo Modules

### 1. Define the Spec (TypeScript)
```tsx
// src/specs/NativeCalculator.ts
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  add(a: number, b: number): number;  // Sync
  multiply(a: number, b: number): Promise<number>;  // Async
  
  // Complex types
  getConstants(): {
    version: string;
    maxValue: number;
  };
}

export default TurboModuleRegistry.getEnforcing<Spec>('Calculator');
```

### 2. Configure Codegen
```json
// package.json
{
  "codegenConfig": {
    "name": "RNCalculatorSpec",
    "type": "modules",
    "jsSrcsDir": "src/specs",
    "android": {
      "javaPackageName": "com.calculator"
    }
  }
}
```

### 3. iOS Implementation (Objective-C++)
```objc
// ios/Calculator.mm
#import "Calculator.h"
#import <RNCalculatorSpec/RNCalculatorSpec.h>

@interface Calculator () <NativeCalculatorSpec>
@end

@implementation Calculator

RCT_EXPORT_MODULE()

- (NSNumber *)add:(double)a b:(double)b {
  return @(a + b);
}

- (void)multiply:(double)a b:(double)b
         resolve:(RCTPromiseResolveBlock)resolve
          reject:(RCTPromiseRejectBlock)reject {
  resolve(@(a * b));
}

- (NSDictionary *)getConstants {
  return @{
    @"version": @"1.0.0",
    @"maxValue": @(1000000)
  };
}

- (std::shared_ptr<facebook::react::TurboModule>)getTurboModule:
    (const facebook::react::ObjCTurboModule::InitParams &)params {
  return std::make_shared<facebook::react::NativeCalculatorSpecJSI>(params);
}

@end
```

### 4. Android Implementation (Kotlin)
```kotlin
// android/src/main/java/com/calculator/CalculatorModule.kt
package com.calculator

import com.facebook.react.bridge.Promise
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.module.annotations.ReactModule

@ReactModule(name = CalculatorModule.NAME)
class CalculatorModule(reactContext: ReactApplicationContext) :
    NativeCalculatorSpec(reactContext) {

    override fun getName() = NAME

    override fun add(a: Double, b: Double): Double = a + b

    override fun multiply(a: Double, b: Double, promise: Promise) {
        promise.resolve(a * b)
    }

    override fun getTypedExportedConstants(): Map<String, Any> = mapOf(
        "version" to "1.0.0",
        "maxValue" to 1000000
    )

    companion object {
        const val NAME = "Calculator"
    }
}
```

## Fabric Components

### Define Component Spec
```tsx
// src/specs/RNCustomViewNativeComponent.ts
import type { ViewProps } from 'react-native';
import type { HostComponent } from 'react-native';
import codegenNativeComponent from 'react-native/Libraries/Utilities/codegenNativeComponent';

interface NativeProps extends ViewProps {
  color?: string;
  radius?: number;
  onPress?: (event: { nativeEvent: { x: number; y: number } }) => void;
}

export default codegenNativeComponent<NativeProps>(
  'RNCustomView'
) as HostComponent<NativeProps>;
```

## Migration Checklist

### Before Migration
- [ ] Update to React Native 0.71+ (0.76+ recommended)
- [ ] Audit native modules for compatibility
- [ ] Check third-party library support
- [ ] Review custom native views

### Migration Steps
1. [ ] Enable New Architecture flag
2. [ ] Run Codegen
3. [ ] Fix TypeScript errors in specs
4. [ ] Update native module implementations
5. [ ] Test thoroughly on both platforms
6. [ ] Profile performance

### Common Issues
- **Library not compatible**: Check library's New Architecture support
- **Codegen errors**: Ensure spec types are correct
- **Native crashes**: Review native code for JSI compatibility
- **Performance regression**: Profile and compare with old architecture

## Interop Layer

React Native provides an interop layer for gradual migration:

```tsx
// Bridge module can still work
import { NativeModules } from 'react-native';
const { LegacyModule } = NativeModules;

// But prefer Turbo Modules
import NativeCalculator from './specs/NativeCalculator';
```

## Best Practices

1. **Start fresh projects with New Architecture** - Default in RN 0.76+
2. **Migrate incrementally** - Use interop layer
3. **Type your specs carefully** - Codegen enforces types
4. **Test on both platforms** - Implementation differences exist
5. **Profile before/after** - Measure actual performance gains
6. **Check library compatibility** - Use react-native-directory
7. **Use synchronous calls sparingly** - Can block JS thread
