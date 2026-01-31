# Mobile Testing with Maestro

Maestro is a simple, powerful E2E testing framework for mobile apps.

## When to Apply

- Writing E2E tests for React Native apps
- Setting up CI/CD mobile testing
- Testing user flows without code
- Quick smoke tests

## Installation

```bash
# macOS/Linux
curl -fsSL "https://get.maestro.mobile.dev" | bash

# macOS via Homebrew
brew tap mobile-dev-inc/tap
brew install mobile-dev-inc/tap/maestro

# Verify
maestro --version
```

**Requires:** Java 17+

## Basic Test Structure

```yaml
# flows/login.yaml
appId: com.myapp
---
- launchApp
- tapOn: "Email"
- inputText: "user@example.com"
- tapOn: "Password"
- inputText: "password123"
- tapOn: "Sign In"
- assertVisible: "Welcome"
```

## Commands Reference

### App Lifecycle
```yaml
- launchApp                    # Launch the app
- launchApp:
    appId: com.other.app       # Launch specific app
    clearState: true           # Clear app data first
- stopApp
- clearState
```

### Tapping
```yaml
- tapOn: "Button Text"         # Tap by text
- tapOn:
    id: "submit-button"        # Tap by testID
- tapOn:
    point: "50%,50%"           # Tap by coordinates
- doubleTapOn: "Item"
- longPressOn: "Delete"
```

### Input
```yaml
- inputText: "Hello World"
- inputText:
    text: "secret"
    label: "Password"          # Input into specific field
- eraseText: 10                # Delete 10 characters
- hideKeyboard
```

### Scrolling
```yaml
- scroll                       # Scroll down
- scrollUntilVisible:
    element: "Load More"
    direction: DOWN
    timeout: 10000
```

### Assertions
```yaml
- assertVisible: "Success"
- assertNotVisible: "Error"
- assertVisible:
    id: "user-avatar"
    timeout: 5000              # Wait up to 5s
```

### Waiting
```yaml
- waitForAnimationToEnd
- extendedWaitUntil:
    visible: "Loaded"
    timeout: 30000
```

### Conditionals
```yaml
- runFlow:
    when:
      visible: "Accept Cookies"
    commands:
      - tapOn: "Accept"
```

## Test Organization

### Project Structure
```
maestro/
├── flows/
│   ├── auth/
│   │   ├── login.yaml
│   │   └── signup.yaml
│   ├── onboarding/
│   │   └── tutorial.yaml
│   └── smoke.yaml
├── config.yaml
└── .maestro/
```

### Reusable Flows
```yaml
# flows/auth/login.yaml
appId: com.myapp
---
- inputText:
    text: ${EMAIL}
    label: "Email"
- inputText:
    text: ${PASSWORD}
    label: "Password"
- tapOn: "Sign In"
```

```yaml
# flows/checkout.yaml
appId: com.myapp
---
- runFlow: auth/login.yaml
- tapOn: "Shop"
- tapOn: "Add to Cart"
```

### Environment Variables
```yaml
# Pass at runtime
# maestro test flows/login.yaml -e EMAIL=test@example.com

- inputText: ${EMAIL}
- inputText: ${PASSWORD:-defaultpass}  # With default
```

## Running Tests

```bash
# Run single flow
maestro test flows/login.yaml

# Run all flows in directory
maestro test flows/

# With environment variables
maestro test -e EMAIL=test@example.com flows/login.yaml

# Record video
maestro record flows/checkout.yaml

# Generate report
maestro test flows/ --format junit --output report.xml
```

## React Native Integration

### Add testID Props
```tsx
<TouchableOpacity testID="submit-button">
  <Text>Submit</Text>
</TouchableOpacity>

<TextInput
  testID="email-input"
  accessibilityLabel="Email"
/>
```

### Test by testID
```yaml
- tapOn:
    id: "submit-button"
- inputText:
    id: "email-input"
    text: "user@example.com"
```

## CI/CD Integration

### GitHub Actions
```yaml
name: E2E Tests

on: [push]

jobs:
  test:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Install Maestro
        run: |
          curl -fsSL "https://get.maestro.mobile.dev" | bash
          echo "$HOME/.maestro/bin" >> $GITHUB_PATH
      
      - name: Start iOS Simulator
        run: |
          xcrun simctl boot "iPhone 15"
      
      - name: Install App
        run: |
          xcrun simctl install booted ./app.app
      
      - name: Run Tests
        run: |
          maestro test flows/ --format junit --output results.xml
      
      - name: Upload Results
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: results.xml
```

## Best Practices

1. **Use testID** - More stable than text selectors
2. **Keep flows small** - One user journey per file
3. **Use reusable flows** - DRY with runFlow
4. **Handle flakiness** - Use appropriate timeouts
5. **Clear state** - Start each test fresh with clearState
6. **Wait for animations** - Use waitForAnimationToEnd
7. **Version control flows** - Treat tests as code
