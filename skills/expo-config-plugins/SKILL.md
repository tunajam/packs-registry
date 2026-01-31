# Expo Config Plugins

Extend Expo's managed workflow with native code modifications.

## When to Apply

- Adding custom native configuration
- Modifying Info.plist or AndroidManifest
- Integrating native SDKs
- Customizing build settings
- Creating reusable native configurations

## Understanding Config Plugins

Config plugins run at build time to modify native project files:
- Info.plist (iOS)
- AndroidManifest.xml (Android)
- Build.gradle (Android)
- Podfile (iOS)
- Xcode project settings

## Using Built-in Plugins

### app.json
```json
{
  "expo": {
    "plugins": [
      "expo-camera",
      "expo-location",
      ["expo-image-picker", {
        "photosPermission": "Allow access to select photos"
      }]
    ]
  }
}
```

### app.config.js (Dynamic)
```js
export default {
  expo: {
    plugins: [
      'expo-camera',
      ['expo-location', {
        locationAlwaysAndWhenInUsePermission: 'Allow $(PRODUCT_NAME) to use your location.',
      }],
    ],
  },
};
```

## Creating Custom Plugins

### Basic Plugin Structure
```js
// plugins/withCustomConfig.js
const { withInfoPlist, withAndroidManifest } = require('@expo/config-plugins');

function withCustomConfig(config, props) {
  // Modify iOS
  config = withInfoPlist(config, (config) => {
    config.modResults.CustomKey = 'CustomValue';
    return config;
  });

  // Modify Android
  config = withAndroidManifest(config, (config) => {
    const mainApplication = config.modResults.manifest.application[0];
    mainApplication.$['android:networkSecurityConfig'] = '@xml/network_security_config';
    return config;
  });

  return config;
}

module.exports = withCustomConfig;
```

### Using the Plugin
```js
// app.config.js
export default {
  expo: {
    plugins: [
      './plugins/withCustomConfig',
      // Or with options
      ['./plugins/withCustomConfig', { someOption: true }],
    ],
  },
};
```

## Common Modifications

### iOS Info.plist
```js
const { withInfoPlist } = require('@expo/config-plugins');

function withIOSConfig(config) {
  return withInfoPlist(config, (config) => {
    // Background modes
    config.modResults.UIBackgroundModes = ['fetch', 'remote-notification'];
    
    // Custom URL schemes
    config.modResults.CFBundleURLTypes = [
      {
        CFBundleURLSchemes: ['myapp'],
      },
    ];
    
    // Permission descriptions
    config.modResults.NSCameraUsageDescription = 'Take photos for your profile';
    
    // Custom settings
    config.modResults.ITSAppUsesNonExemptEncryption = false;
    
    return config;
  });
}

module.exports = withIOSConfig;
```

### Android Manifest
```js
const { withAndroidManifest } = require('@expo/config-plugins');

function withAndroidConfig(config) {
  return withAndroidManifest(config, (config) => {
    const manifest = config.modResults.manifest;
    
    // Add permissions
    manifest['uses-permission'] = manifest['uses-permission'] || [];
    manifest['uses-permission'].push({
      $: { 'android:name': 'android.permission.VIBRATE' },
    });
    
    // Modify application
    const application = manifest.application[0];
    application.$['android:largeHeap'] = 'true';
    
    // Add meta-data
    application['meta-data'] = application['meta-data'] || [];
    application['meta-data'].push({
      $: {
        'android:name': 'com.google.firebase.messaging.default_notification_icon',
        'android:resource': '@drawable/notification_icon',
      },
    });
    
    return config;
  });
}

module.exports = withAndroidConfig;
```

### Android Gradle
```js
const { withAppBuildGradle } = require('@expo/config-plugins');

function withAndroidGradle(config) {
  return withAppBuildGradle(config, (config) => {
    config.modResults.contents = config.modResults.contents.replace(
      'dependencies {',
      `dependencies {
    implementation 'com.some.library:version'`
    );
    return config;
  });
}

module.exports = withAndroidGradle;
```

### iOS Podfile
```js
const { withPodfile } = require('@expo/config-plugins');

function withIOSPodfile(config) {
  return withPodfile(config, (config) => {
    config.modResults.contents = config.modResults.contents.replace(
      "post_install do |installer|",
      `
  pod 'SomeNativePod', '~> 1.0'
  
  post_install do |installer|`
    );
    return config;
  });
}

module.exports = withIOSPodfile;
```

## Xcode Project Modifications

### Build Settings
```js
const { withXcodeProject } = require('@expo/config-plugins');

function withXcodeBuildSettings(config) {
  return withXcodeProject(config, (config) => {
    const project = config.modResults;
    
    project.addBuildProperty('SWIFT_VERSION', '5.0');
    project.addBuildProperty('ENABLE_BITCODE', 'NO');
    
    return config;
  });
}

module.exports = withXcodeBuildSettings;
```

### Add Files
```js
const { withXcodeProject, IOSConfig } = require('@expo/config-plugins');
const path = require('path');

function withCustomFiles(config) {
  return withXcodeProject(config, (config) => {
    const project = config.modResults;
    const projectRoot = config.modRequest.projectRoot;
    
    // Add a file to the project
    const filePath = path.join(projectRoot, 'custom-file.swift');
    project.addSourceFile(filePath, {}, 'Resources');
    
    return config;
  });
}

module.exports = withCustomFiles;
```

## Plugin Chaining

```js
const { withPlugins } = require('@expo/config-plugins');
const withIOSConfig = require('./withIOSConfig');
const withAndroidConfig = require('./withAndroidConfig');

function withMyApp(config, props) {
  return withPlugins(config, [
    [withIOSConfig, props],
    [withAndroidConfig, props],
  ]);
}

module.exports = withMyApp;
```

## Prebuild Command

```bash
# Generate native projects
npx expo prebuild

# Clean and regenerate
npx expo prebuild --clean

# Platform specific
npx expo prebuild --platform ios
```

## TypeScript Plugins

```ts
// plugins/withCustomConfig.ts
import { ConfigPlugin, withInfoPlist } from '@expo/config-plugins';

interface Props {
  apiKey: string;
}

const withCustomConfig: ConfigPlugin<Props> = (config, { apiKey }) => {
  return withInfoPlist(config, (config) => {
    config.modResults.API_KEY = apiKey;
    return config;
  });
};

export default withCustomConfig;
```

## Common Plugin Patterns

### Add Deep Link Scheme
```js
const { withInfoPlist, withAndroidManifest } = require('@expo/config-plugins');

function withDeepLinks(config, { scheme }) {
  // iOS
  config = withInfoPlist(config, (config) => {
    const existing = config.modResults.CFBundleURLTypes || [];
    config.modResults.CFBundleURLTypes = [
      ...existing,
      { CFBundleURLSchemes: [scheme] },
    ];
    return config;
  });

  // Android
  config = withAndroidManifest(config, (config) => {
    const activity = config.modResults.manifest.application[0].activity[0];
    activity['intent-filter'] = activity['intent-filter'] || [];
    activity['intent-filter'].push({
      action: [{ $: { 'android:name': 'android.intent.action.VIEW' } }],
      category: [
        { $: { 'android:name': 'android.intent.category.DEFAULT' } },
        { $: { 'android:name': 'android.intent.category.BROWSABLE' } },
      ],
      data: [{ $: { 'android:scheme': scheme } }],
    });
    return config;
  });

  return config;
}

module.exports = withDeepLinks;
```

## Best Practices

1. **Keep plugins focused** - One modification per plugin
2. **Make plugins configurable** - Accept options for flexibility
3. **Test with prebuild** - Verify changes before building
4. **Document modifications** - Others need to understand changes
5. **Use TypeScript** - Better type safety and IDE support
6. **Handle existing values** - Don't overwrite, merge
7. **Check for null** - Native structures can be incomplete
