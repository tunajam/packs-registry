# E2E Testing with Detox

Detox is a gray-box E2E testing framework for React Native apps by Wix.

## When to Apply

- Writing comprehensive E2E tests
- Testing complex async operations
- CI/CD pipeline testing
- Testing native interactions

## Installation

```bash
# Install Detox CLI
npm install -g detox-cli

# Install Detox in project
npm install detox --save-dev

# iOS: Install applesimutils
brew tap wix/brew
brew install applesimutils

# Initialize Detox config
detox init
```

## Configuration

### .detoxrc.js
```js
module.exports = {
  testRunner: {
    args: {
      $0: 'jest',
      config: 'e2e/jest.config.js',
    },
    jest: {
      setupTimeout: 120000,
    },
  },
  apps: {
    'ios.debug': {
      type: 'ios.app',
      binaryPath: 'ios/build/Build/Products/Debug-iphonesimulator/MyApp.app',
      build: 'xcodebuild -workspace ios/MyApp.xcworkspace -scheme MyApp -configuration Debug -sdk iphonesimulator -derivedDataPath ios/build',
    },
    'ios.release': {
      type: 'ios.app',
      binaryPath: 'ios/build/Build/Products/Release-iphonesimulator/MyApp.app',
      build: 'xcodebuild -workspace ios/MyApp.xcworkspace -scheme MyApp -configuration Release -sdk iphonesimulator -derivedDataPath ios/build',
    },
    'android.debug': {
      type: 'android.apk',
      binaryPath: 'android/app/build/outputs/apk/debug/app-debug.apk',
      build: 'cd android && ./gradlew assembleDebug assembleAndroidTest -DtestBuildType=debug',
      reversePorts: [8081],
    },
    'android.release': {
      type: 'android.apk',
      binaryPath: 'android/app/build/outputs/apk/release/app-release.apk',
      build: 'cd android && ./gradlew assembleRelease assembleAndroidTest -DtestBuildType=release',
    },
  },
  devices: {
    simulator: {
      type: 'ios.simulator',
      device: { type: 'iPhone 15' },
    },
    emulator: {
      type: 'android.emulator',
      device: { avdName: 'Pixel_4_API_33' },
    },
  },
  configurations: {
    'ios.sim.debug': {
      device: 'simulator',
      app: 'ios.debug',
    },
    'ios.sim.release': {
      device: 'simulator',
      app: 'ios.release',
    },
    'android.emu.debug': {
      device: 'emulator',
      app: 'android.debug',
    },
    'android.emu.release': {
      device: 'emulator',
      app: 'android.release',
    },
  },
};
```

### Jest Config (e2e/jest.config.js)
```js
module.exports = {
  rootDir: '..',
  testMatch: ['<rootDir>/e2e/**/*.test.js'],
  testTimeout: 120000,
  maxWorkers: 1,
  globalSetup: 'detox/runners/jest/globalSetup',
  globalTeardown: 'detox/runners/jest/globalTeardown',
  reporters: ['detox/runners/jest/reporter'],
  testEnvironment: 'detox/runners/jest/testEnvironment',
  verbose: true,
};
```

## Writing Tests

### Basic Test Structure
```js
describe('Login Flow', () => {
  beforeAll(async () => {
    await device.launchApp();
  });

  beforeEach(async () => {
    await device.reloadReactNative();
  });

  it('should show login screen', async () => {
    await expect(element(by.id('login-screen'))).toBeVisible();
  });

  it('should login successfully', async () => {
    await element(by.id('email-input')).typeText('user@example.com');
    await element(by.id('password-input')).typeText('password123');
    await element(by.id('login-button')).tap();
    
    await expect(element(by.id('home-screen'))).toBeVisible();
  });
});
```

### Matchers

```js
// By testID
element(by.id('submit-button'))

// By text
element(by.text('Sign In'))

// By label (accessibility)
element(by.label('Close'))

// By type
element(by.type('RCTTextInput'))

// By traits (iOS)
element(by.traits(['button']))

// Combining matchers
element(by.id('item').withAncestor(by.id('list')))
element(by.id('icon').withDescendant(by.text('Home')))
```

### Actions

```js
// Tapping
await element(by.id('button')).tap();
await element(by.id('button')).longPress();
await element(by.id('button')).multiTap(3);

// Typing
await element(by.id('input')).typeText('Hello');
await element(by.id('input')).replaceText('World');
await element(by.id('input')).clearText();

// Scrolling
await element(by.id('scrollView')).scroll(200, 'down');
await element(by.id('scrollView')).scrollTo('bottom');
await element(by.id('list')).scrollToIndex(10);

// Swiping
await element(by.id('card')).swipe('left');
await element(by.id('carousel')).swipe('right', 'fast', 0.75);

// Pinch
await element(by.id('map')).pinch(1.5); // zoom in
await element(by.id('map')).pinch(0.5); // zoom out
```

### Expectations

```js
await expect(element(by.id('title'))).toBeVisible();
await expect(element(by.id('error'))).not.toBeVisible();
await expect(element(by.id('button'))).toExist();
await expect(element(by.id('input'))).toHaveText('Hello');
await expect(element(by.id('input'))).toHaveValue('World');
await expect(element(by.id('switch'))).toHaveToggleValue(true);
await expect(element(by.id('label'))).toHaveLabel('Submit');
```

### Device Actions

```js
// App lifecycle
await device.launchApp({ newInstance: true });
await device.reloadReactNative();
await device.terminateApp();
await device.installApp();
await device.uninstallApp();

// Device control
await device.sendToHome();
await device.shake();
await device.setLocation(32.0853, 34.7818);
await device.setURLBlacklist(['.*google.com.*']);

// Permissions (iOS)
await device.launchApp({
  permissions: { notifications: 'YES', photos: 'YES' }
});
```

## React Native Setup

### Add testID Props
```tsx
<TouchableOpacity testID="submit-button">
  <Text>Submit</Text>
</TouchableOpacity>

<TextInput
  testID="email-input"
  placeholder="Email"
/>

<FlatList
  testID="items-list"
  data={items}
  renderItem={({ item, index }) => (
    <View testID={`item-${index}`}>
      <Text>{item.name}</Text>
    </View>
  )}
/>
```

## Running Tests

```bash
# Build for testing
detox build --configuration ios.sim.debug

# Run tests
detox test --configuration ios.sim.debug

# Run specific test file
detox test --configuration ios.sim.debug e2e/login.test.js

# With recording on failure
detox test --configuration ios.sim.debug --record-videos failing

# Generate JUnit report
detox test --configuration ios.sim.debug --jest-report-specs
```

## Best Practices

1. **Use testID consistently** - Every interactive element needs one
2. **Keep tests independent** - Each test should work in isolation
3. **Wait properly** - Detox auto-syncs, but add explicit waits if needed
4. **Test on both platforms** - iOS and Android can behave differently
5. **Clean state** - Use `device.launchApp({ delete: true })` for fresh starts
6. **Mock network** - Use `device.setURLBlacklist()` or mock servers
7. **Parallel execution** - Use multiple workers for faster CI runs
