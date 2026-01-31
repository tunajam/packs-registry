# EAS Build

Build and deploy Expo apps with Expo Application Services.

## When to Apply

- Building production apps
- Setting up CI/CD pipelines
- Managing app credentials
- Creating development builds
- Configuring build profiles

## Setup

```bash
# Install EAS CLI
npm install -g eas-cli

# Login to Expo account
eas login

# Configure project
eas build:configure
```

## eas.json Configuration

```json
{
  "cli": {
    "version": ">= 12.0.0"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "ios": {
        "simulator": true
      }
    },
    "preview": {
      "distribution": "internal",
      "android": {
        "buildType": "apk"
      }
    },
    "production": {
      "autoIncrement": true
    }
  },
  "submit": {
    "production": {}
  }
}
```

## Build Profiles

### Development Build
```json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "env": {
        "APP_ENV": "development"
      },
      "ios": {
        "simulator": true
      },
      "android": {
        "gradleCommand": ":app:assembleDebug"
      }
    },
    "development-device": {
      "extends": "development",
      "ios": {
        "simulator": false
      }
    }
  }
}
```

### Preview Build
```json
{
  "build": {
    "preview": {
      "distribution": "internal",
      "env": {
        "APP_ENV": "staging"
      },
      "channel": "preview",
      "android": {
        "buildType": "apk"
      },
      "ios": {
        "enterpriseProvisioning": "universal"
      }
    }
  }
}
```

### Production Build
```json
{
  "build": {
    "production": {
      "env": {
        "APP_ENV": "production"
      },
      "channel": "production",
      "autoIncrement": true,
      "android": {
        "buildType": "app-bundle"
      },
      "ios": {
        "autoIncrement": "buildNumber"
      }
    }
  }
}
```

## Build Commands

```bash
# Development build (simulator)
eas build --profile development --platform ios

# Development build (device)
eas build --profile development-device --platform ios

# Preview build (internal testing)
eas build --profile preview --platform all

# Production build
eas build --profile production --platform all

# Run build locally (requires native setup)
eas build --local --profile development --platform ios

# Check build status
eas build:list

# View build logs
eas build:view
```

## Environment Variables

### In eas.json
```json
{
  "build": {
    "production": {
      "env": {
        "API_URL": "https://api.production.com",
        "SENTRY_DSN": "@sentry-dsn"
      }
    }
  }
}
```

### Using Secrets
```bash
# Set secret
eas secret:create --name SENTRY_DSN --value "your-dsn-value"

# List secrets
eas secret:list

# Reference in eas.json
{
  "build": {
    "production": {
      "env": {
        "SENTRY_DSN": "@SENTRY_DSN"
      }
    }
  }
}
```

### In app.config.js
```js
export default {
  expo: {
    extra: {
      apiUrl: process.env.API_URL,
      eas: {
        projectId: 'your-project-id',
      },
    },
  },
};
```

## Credentials Management

### iOS Credentials
```bash
# Setup credentials (guided)
eas credentials --platform ios

# Generate new credentials
eas build --platform ios --auto-submit

# Use existing credentials
eas credentials configure
```

### Android Credentials
```bash
# Setup keystore
eas credentials --platform android

# Download keystore
eas credentials --platform android
# Select "Download keystore"
```

### Managing Provisioning Profiles
```json
{
  "build": {
    "production": {
      "ios": {
        "credentialsSource": "local",
        "provisioningProfilePath": "./certs/profile.mobileprovision",
        "distributionCertificate": {
          "path": "./certs/dist.p12",
          "password": "@CERT_PASSWORD"
        }
      }
    }
  }
}
```

## Internal Distribution

### iOS Ad Hoc
```json
{
  "build": {
    "preview": {
      "distribution": "internal",
      "ios": {
        "enterpriseProvisioning": "adhoc"
      }
    }
  }
}
```

### Android APK
```json
{
  "build": {
    "preview": {
      "android": {
        "buildType": "apk"
      }
    }
  }
}
```

### Register Test Devices
```bash
# Register device (opens QR code)
eas device:create

# List registered devices
eas device:list
```

## Version Management

### Auto Increment
```json
{
  "build": {
    "production": {
      "autoIncrement": true
    }
  }
}
```

### Manual Version Control
```json
// app.json
{
  "expo": {
    "version": "1.0.0",
    "ios": {
      "buildNumber": "1"
    },
    "android": {
      "versionCode": 1
    }
  }
}
```

### Remote Version Source
```json
{
  "build": {
    "production": {
      "ios": {
        "autoIncrement": "buildNumber"
      },
      "android": {
        "autoIncrement": "versionCode"
      }
    }
  }
}
```

## Build Hooks

### Pre-build script
```json
{
  "build": {
    "production": {
      "prebuildCommand": "npm run generate-assets"
    }
  }
}
```

### Custom Xcode project config
```js
// app.config.js
export default {
  expo: {
    ios: {
      infoPlist: {
        UIBackgroundModes: ['fetch', 'remote-notification'],
      },
    },
  },
};
```

## CI/CD Integration

### GitHub Actions
```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      
      - name: Install dependencies
        run: npm ci
      
      - name: Setup EAS
        uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}
      
      - name: Build app
        run: eas build --platform all --profile production --non-interactive
      
      - name: Submit to stores
        run: eas submit --platform all --profile production --non-interactive
```

### Required Secrets
```
EXPO_TOKEN - Generate at expo.dev/settings/access-tokens
```

## Submission

### App Store Connect
```bash
# Submit iOS build
eas submit --platform ios --latest

# Or specific build
eas submit --platform ios --id BUILD_ID
```

### Google Play Store
```bash
# Submit Android build
eas submit --platform android --latest

# With track
eas submit --platform android --latest --track internal
```

### eas.json Submit Config
```json
{
  "submit": {
    "production": {
      "ios": {
        "appleId": "your@email.com",
        "ascAppId": "123456789",
        "appleTeamId": "TEAM_ID"
      },
      "android": {
        "serviceAccountKeyPath": "./play-store-key.json",
        "track": "production"
      }
    }
  }
}
```

## Best Practices

1. **Use separate profiles** - Development, preview, production
2. **Store secrets securely** - Use EAS Secrets, not env files
3. **Auto-increment versions** - Let EAS manage build numbers
4. **Test preview builds** - Before submitting to stores
5. **Set up CI/CD** - Automate builds on merge to main
6. **Monitor build times** - Optimize with caching
7. **Keep credentials secure** - Use EAS managed credentials
