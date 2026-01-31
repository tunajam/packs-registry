# React Native Deep Linking

Configure deep links and universal links for mobile apps.

## When to Apply

- Opening app from URLs
- Email/SMS app launch links
- Social sharing integration
- Web to app handoff
- Marketing campaign links

## Types of Links

| Type | iOS | Android | Example |
|------|-----|---------|---------|
| Deep Link | URL Scheme | Custom Scheme | `myapp://profile/123` |
| Universal Link | Associated Domains | App Links | `https://myapp.com/profile/123` |

## Deep Links (URL Schemes)

### iOS Configuration

```xml
<!-- ios/MyApp/Info.plist -->
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>myapp</string>
    </array>
    <key>CFBundleURLName</key>
    <string>com.mycompany.myapp</string>
  </dict>
</array>
```

### Android Configuration

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<activity
  android:name=".MainActivity"
  android:launchMode="singleTask">
  
  <intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="myapp" />
  </intent-filter>
  
</activity>
```

### Expo Configuration

```json
// app.json
{
  "expo": {
    "scheme": "myapp",
    "ios": {
      "bundleIdentifier": "com.mycompany.myapp"
    },
    "android": {
      "package": "com.mycompany.myapp",
      "intentFilters": [
        {
          "action": "VIEW",
          "data": [{ "scheme": "myapp" }],
          "category": ["BROWSABLE", "DEFAULT"]
        }
      ]
    }
  }
}
```

## Universal Links (iOS) / App Links (Android)

### iOS Associated Domains

```json
// app.json (Expo)
{
  "expo": {
    "ios": {
      "associatedDomains": ["applinks:myapp.com"]
    }
  }
}
```

```xml
<!-- Or in Xcode Entitlements -->
<key>com.apple.developer.associated-domains</key>
<array>
  <string>applinks:myapp.com</string>
  <string>applinks:www.myapp.com</string>
</array>
```

### Apple App Site Association

Host at: `https://myapp.com/.well-known/apple-app-site-association`

```json
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "TEAM_ID.com.mycompany.myapp",
        "paths": ["/profile/*", "/product/*", "/share/*"]
      }
    ]
  }
}
```

### Android App Links

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<intent-filter android:autoVerify="true">
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data 
    android:scheme="https"
    android:host="myapp.com"
    android:pathPrefix="/profile" />
</intent-filter>
```

### Digital Asset Links (Android)

Host at: `https://myapp.com/.well-known/assetlinks.json`

```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.mycompany.myapp",
    "sha256_cert_fingerprints": [
      "AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99"
    ]
  }
}]
```

Get SHA256:
```bash
keytool -list -v -keystore my-release-key.keystore -alias my-key-alias
```

## Handling Links in React Native

### Linking API
```tsx
import { Linking } from 'react-native';

// Get initial URL (app opened via link)
const url = await Linking.getInitialURL();

// Listen for incoming links
useEffect(() => {
  const subscription = Linking.addEventListener('url', ({ url }) => {
    handleDeepLink(url);
  });

  // Check initial URL
  Linking.getInitialURL().then((url) => {
    if (url) handleDeepLink(url);
  });

  return () => subscription.remove();
}, []);

function handleDeepLink(url: string) {
  const parsed = parseUrl(url);
  // Navigate based on parsed URL
}
```

### With React Navigation
```tsx
import { NavigationContainer } from '@react-navigation/native';

const linking = {
  prefixes: ['myapp://', 'https://myapp.com'],
  config: {
    screens: {
      Home: '',
      Profile: 'profile/:userId',
      Product: 'product/:productId',
      Settings: 'settings',
      NotFound: '*',
    },
  },
};

function App() {
  return (
    <NavigationContainer 
      linking={linking} 
      fallback={<LoadingScreen />}
    >
      <RootNavigator />
    </NavigationContainer>
  );
}
```

### Nested Navigation
```tsx
const linking = {
  prefixes: ['myapp://', 'https://myapp.com'],
  config: {
    screens: {
      Main: {
        screens: {
          Home: '',
          Profile: {
            path: 'profile/:userId',
            screens: {
              ProfileDetails: '',
              ProfileSettings: 'settings',
            },
          },
        },
      },
      Modal: 'modal',
    },
  },
};
```

### Custom Path Parsing
```tsx
const linking = {
  prefixes: ['myapp://'],
  config: {
    screens: {
      Product: {
        path: 'product/:id',
        parse: {
          id: (id: string) => `product-${id}`,
        },
        stringify: {
          id: (id: string) => id.replace('product-', ''),
        },
      },
    },
  },
};
```

## Expo Linking

```tsx
import * as Linking from 'expo-linking';

// Get deep link URL
const url = Linking.createURL('profile/123');
// Development: exp://192.168.1.5:8081/--/profile/123
// Production: myapp://profile/123

// Parse URL
const { path, queryParams } = Linking.parse(url);

// Open URL
await Linking.openURL('https://example.com');
await Linking.openURL('tel:+1234567890');
await Linking.openURL('mailto:test@example.com');
```

## Testing Deep Links

### iOS Simulator
```bash
xcrun simctl openurl booted "myapp://profile/123"
```

### Android Emulator
```bash
adb shell am start -W -a android.intent.action.VIEW -d "myapp://profile/123" com.mycompany.myapp
```

### Expo
```bash
# In development
npx uri-scheme open "myapp://profile/123" --ios
npx uri-scheme open "myapp://profile/123" --android
```

## Deferred Deep Links

For links clicked before app installation:

### Using Branch
```tsx
import branch from 'react-native-branch';

branch.subscribe(({ error, params, uri }) => {
  if (error) {
    console.error('Branch error: ' + error);
    return;
  }
  
  if (params['+clicked_branch_link']) {
    // Handle deferred deep link
    const { productId } = params;
    navigation.navigate('Product', { productId });
  }
});
```

### Using Firebase Dynamic Links
```tsx
import dynamicLinks from '@react-native-firebase/dynamic-links';

// Get initial link
const initialLink = await dynamicLinks().getInitialLink();
if (initialLink) {
  handleDynamicLink(initialLink);
}

// Listen for links
const unsubscribe = dynamicLinks().onLink(handleDynamicLink);

function handleDynamicLink(link) {
  const url = link.url;
  // Parse and navigate
}
```

## Common Patterns

### Share URLs
```tsx
import { Share } from 'react-native';

async function shareProfile(userId: string) {
  await Share.share({
    message: 'Check out this profile!',
    url: `https://myapp.com/profile/${userId}`,
  });
}
```

### Authentication Callback
```tsx
// Configure OAuth redirect
const redirectUri = Linking.createURL('auth/callback');

// Handle callback
const linking = {
  config: {
    screens: {
      AuthCallback: {
        path: 'auth/callback',
        parse: {
          code: (code: string) => code,
        },
      },
    },
  },
};
```

## Best Practices

1. **Use universal links** - More reliable than schemes
2. **Handle all states** - Cold start, background, foreground
3. **Validate URLs** - Don't trust incoming data
4. **Test thoroughly** - All link types and scenarios
5. **Provide fallbacks** - Web fallback for non-installed users
6. **Use consistent paths** - Same structure as web app
7. **Document your links** - For marketing team
