# React Native Mobile Security

Security best practices for React Native applications.

## When to Apply

- Storing sensitive data
- Implementing authentication
- Protecting API communications
- Securing app code
- Handling PII

## Secure Storage

### Never Use AsyncStorage for Secrets
```tsx
// ❌ WRONG - AsyncStorage is NOT encrypted
import AsyncStorage from '@react-native-async-storage/async-storage';
await AsyncStorage.setItem('authToken', token); // INSECURE

// ✅ CORRECT - Use secure storage
```

### Expo SecureStore
```tsx
import * as SecureStore from 'expo-secure-store';

// Store
await SecureStore.setItemAsync('authToken', token);

// Retrieve
const token = await SecureStore.getItemAsync('authToken');

// Delete
await SecureStore.deleteItemAsync('authToken');

// Options
await SecureStore.setItemAsync('biometricToken', token, {
  keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
  requireAuthentication: true, // Biometric required
});
```

### react-native-keychain
```tsx
import * as Keychain from 'react-native-keychain';

// Store credentials
await Keychain.setGenericPassword(username, password, {
  service: 'com.myapp.auth',
  accessControl: Keychain.ACCESS_CONTROL.BIOMETRY_ANY,
  accessible: Keychain.ACCESSIBLE.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
});

// Retrieve
const credentials = await Keychain.getGenericPassword({ service: 'com.myapp.auth' });
if (credentials) {
  console.log(credentials.username, credentials.password);
}

// With biometric prompt
const credentials = await Keychain.getGenericPassword({
  authenticationPrompt: {
    title: 'Authenticate to access your account',
    cancel: 'Cancel',
  },
});

// Clear
await Keychain.resetGenericPassword({ service: 'com.myapp.auth' });
```

## SSL/Certificate Pinning

### Why Pin Certificates?
Prevents MITM attacks even if attacker has valid CA certificate.

### react-native-ssl-pinning
```tsx
import { fetch } from 'react-native-ssl-pinning';

// Using certificate
const response = await fetch('https://api.example.com/data', {
  method: 'GET',
  headers: {
    'Content-Type': 'application/json',
  },
  sslPinning: {
    certs: ['cert1', 'cert2'], // Certificate names in native folder
  },
});

// Using public key hash
const response = await fetch('https://api.example.com/data', {
  sslPinning: {
    certs: ['sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA='],
  },
});
```

### Get Certificate Hash
```bash
# Get hash for pinning
openssl s_client -connect api.example.com:443 2>/dev/null | \
  openssl x509 -pubkey -noout | \
  openssl pkey -pubin -outform der | \
  openssl dgst -sha256 -binary | \
  openssl enc -base64
```

### TrustKit (iOS)
```xml
<!-- ios/MyApp/Info.plist -->
<key>TSKConfiguration</key>
<dict>
  <key>TSKPinnedDomains</key>
  <dict>
    <key>api.example.com</key>
    <dict>
      <key>TSKPublicKeyHashes</key>
      <array>
        <string>BASE64_HASH_1</string>
        <string>BASE64_HASH_2</string>
      </array>
      <key>TSKEnforcePinning</key>
      <true/>
    </dict>
  </dict>
</dict>
```

## Code Obfuscation

### Hermes Bytecode (Default)
```json
// app.json (Expo)
{
  "expo": {
    "jsEngine": "hermes"
  }
}
```
Hermes compiles to bytecode, not readable JS.

### ProGuard (Android)
```gradle
// android/app/build.gradle
android {
  buildTypes {
    release {
      minifyEnabled true
      shrinkResources true
      proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
    }
  }
}
```

```pro
# android/app/proguard-rules.pro
-keep class com.myapp.** { *; }
-keep class com.facebook.hermes.unicode.** { *; }
-keep class com.facebook.jni.** { *; }
```

### JavaScript Obfuscation
```bash
# Using metro config
npm install metro-minify-terser

# metro.config.js
module.exports = {
  transformer: {
    minifierPath: 'metro-minify-terser',
    minifierConfig: {
      mangle: {
        toplevel: true,
      },
      compress: {
        drop_console: true,
        drop_debugger: true,
      },
    },
  },
};
```

## Authentication Security

### Secure Token Storage
```tsx
// auth.ts
import * as SecureStore from 'expo-secure-store';

const TOKEN_KEY = 'auth_token';
const REFRESH_KEY = 'refresh_token';

export async function storeTokens(access: string, refresh: string) {
  await SecureStore.setItemAsync(TOKEN_KEY, access);
  await SecureStore.setItemAsync(REFRESH_KEY, refresh);
}

export async function getAccessToken() {
  return await SecureStore.getItemAsync(TOKEN_KEY);
}

export async function clearTokens() {
  await SecureStore.deleteItemAsync(TOKEN_KEY);
  await SecureStore.deleteItemAsync(REFRESH_KEY);
}
```

### Biometric Authentication
```tsx
import * as LocalAuthentication from 'expo-local-authentication';

async function authenticateWithBiometrics() {
  // Check hardware support
  const hasHardware = await LocalAuthentication.hasHardwareAsync();
  if (!hasHardware) return false;

  // Check enrollment
  const isEnrolled = await LocalAuthentication.isEnrolledAsync();
  if (!isEnrolled) return false;

  // Authenticate
  const result = await LocalAuthentication.authenticateAsync({
    promptMessage: 'Authenticate to continue',
    fallbackLabel: 'Use passcode',
    disableDeviceFallback: false,
  });

  return result.success;
}
```

## Network Security

### Android Network Security Config
```xml
<!-- android/app/src/main/res/xml/network_security_config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
  <!-- Production: no cleartext, certificate pinning -->
  <domain-config cleartextTrafficPermitted="false">
    <domain includeSubdomains="true">api.example.com</domain>
    <pin-set expiration="2025-01-01">
      <pin digest="SHA-256">BASE64_HASH</pin>
      <!-- Backup pin -->
      <pin digest="SHA-256">BACKUP_HASH</pin>
    </pin-set>
  </domain-config>
  
  <!-- Block all other cleartext -->
  <base-config cleartextTrafficPermitted="false" />
</network-security-config>
```

```xml
<!-- AndroidManifest.xml -->
<application
  android:networkSecurityConfig="@xml/network_security_config"
  ...>
```

### iOS App Transport Security
```xml
<!-- ios/MyApp/Info.plist -->
<key>NSAppTransportSecurity</key>
<dict>
  <key>NSAllowsArbitraryLoads</key>
  <false/>
  <!-- Only if needed for specific domains -->
  <key>NSExceptionDomains</key>
  <dict>
    <key>legacy-api.example.com</key>
    <dict>
      <key>NSExceptionAllowsInsecureHTTPLoads</key>
      <true/>
    </dict>
  </dict>
</dict>
```

## Data Protection

### Encryption at Rest
```tsx
import CryptoES from 'crypto-es';

// Encrypt data
function encrypt(data: string, key: string): string {
  return CryptoES.AES.encrypt(data, key).toString();
}

// Decrypt data
function decrypt(ciphertext: string, key: string): string {
  const bytes = CryptoES.AES.decrypt(ciphertext, key);
  return bytes.toString(CryptoES.enc.Utf8);
}

// Usage
const encrypted = encrypt(JSON.stringify(sensitiveData), masterKey);
await SecureStore.setItemAsync('encrypted_data', encrypted);
```

### Secure Random
```tsx
import * as Crypto from 'expo-crypto';

// Generate secure random bytes
const randomBytes = await Crypto.getRandomBytesAsync(32);

// Generate UUID
const uuid = Crypto.randomUUID();
```

## Runtime Protection

### Jailbreak/Root Detection
```tsx
import JailMonkey from 'jail-monkey';

function checkDeviceIntegrity() {
  if (JailMonkey.isJailBroken()) {
    // Device is jailbroken/rooted
    // Warn user or restrict functionality
    return false;
  }
  
  if (JailMonkey.isDebuggedMode()) {
    // Debugger attached
    return false;
  }
  
  return true;
}
```

### Screen Capture Prevention
```tsx
// iOS: Prevent screenshots
import { usePreventScreenCapture } from 'expo-screen-capture';

function SensitiveScreen() {
  usePreventScreenCapture();
  return <View>...</View>;
}

// Android: Secure flag
// MainActivity.java
getWindow().setFlags(
  WindowManager.LayoutParams.FLAG_SECURE,
  WindowManager.LayoutParams.FLAG_SECURE
);
```

## Security Checklist

### Storage
- [ ] Never store secrets in AsyncStorage
- [ ] Use Keychain/Keystore for credentials
- [ ] Encrypt sensitive local data
- [ ] Clear data on logout

### Network
- [ ] Use HTTPS only
- [ ] Implement certificate pinning
- [ ] Remove debug endpoints
- [ ] Validate all server responses

### Code
- [ ] Enable Hermes (bytecode)
- [ ] Enable ProGuard (Android)
- [ ] Remove console.log in production
- [ ] Obfuscate sensitive logic

### Authentication
- [ ] Secure token storage
- [ ] Token expiration handling
- [ ] Biometric option for sensitive actions
- [ ] Session timeout

### Runtime
- [ ] Jailbreak/root detection
- [ ] Debugger detection
- [ ] Screenshot prevention for sensitive screens
- [ ] Clipboard handling
