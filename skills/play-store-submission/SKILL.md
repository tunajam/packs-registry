# Google Play Store Submission

Complete guide to submitting apps to the Google Play Store.

## When to Apply

- First Play Store submission
- Understanding Play Store policies
- Setting up releases and tracks
- Managing app bundle delivery
- Handling policy violations

## Prerequisites

- Google Play Developer account ($25 one-time)
- Google Play Console access
- Signed release AAB/APK
- Privacy policy URL

## Play Console Setup

### Create App
1. Go to Google Play Console
2. Create app > Fill details:
   - App name
   - Default language
   - App or Game
   - Free or Paid

### Store Listing

```yaml
Main Store Listing:
  App name: 30 characters max
  Short description: 80 characters max
  Full description: 4000 characters max

Graphics:
  App icon: 512x512 PNG (no alpha)
  Feature graphic: 1024x500
  Phone screenshots: 2-8 required
  Tablet screenshots: Required if supporting tablets
  Promo video: YouTube URL (optional)

Contact Details:
  Email: Required
  Phone: Optional
  Website: Optional

Privacy Policy: Required URL
```

## Screenshots Requirements

```yaml
Phone:
  - Minimum: 2 screenshots
  - Maximum: 8 screenshots
  - Size: 16:9 or 9:16 aspect ratio
  - Min: 320px, Max: 3840px
  - PNG or JPEG (no alpha)

7-inch Tablet:
  - Required if targeting tablets
  - Same specs as phone

10-inch Tablet:
  - Required if targeting tablets
  - Same specs as phone
```

## App Bundle / APK

### Android App Bundle (Required)
```bash
# Build AAB
cd android && ./gradlew bundleRelease

# Output: android/app/build/outputs/bundle/release/app-release.aab
```

### Signing
```bash
# Generate keystore (one-time)
keytool -genkey -v -keystore my-upload-key.keystore \
  -alias my-key-alias -keyalg RSA -keysize 2048 -validity 10000

# Configure gradle
# android/gradle.properties
MYAPP_UPLOAD_STORE_FILE=my-upload-key.keystore
MYAPP_UPLOAD_KEY_ALIAS=my-key-alias
MYAPP_UPLOAD_STORE_PASSWORD=*****
MYAPP_UPLOAD_KEY_PASSWORD=*****
```

### Using EAS (Expo)
```bash
# Build for Play Store
eas build --platform android --profile production

# Submit
eas submit --platform android
```

## Release Tracks

```yaml
Internal Testing:
  - Up to 100 testers
  - No review required
  - Instant availability
  - Great for QA team

Closed Testing:
  - Invite-only via email list
  - Brief review (hours)
  - Good for beta testers

Open Testing:
  - Anyone can join via link
  - Brief review
  - Public beta

Production:
  - Full review required
  - Available to everyone
  - Staged rollouts supported
```

### Staged Rollout
```yaml
Production release options:
  - Full rollout: 100% immediately
  - Staged rollout:
    - Start at 5%, 10%, 20%, 50%, 100%
    - Monitor crashes/ANRs
    - Pause or halt if issues
    - Can't decrease percentage
```

## Content Rating

### Complete Questionnaire
1. Go to Policy > App content > Content rating
2. Answer questions about:
   - Violence
   - Sexuality
   - Language
   - Controlled substances
   - User interaction
3. Get ratings for all regions

## Data Safety Section

```yaml
Required declarations:
  Data collection:
    - What data you collect
    - How it's used
    - Is it shared with third parties
    
  Security practices:
    - Data encrypted in transit
    - Data deletion available
    
  Data types:
    - Personal info
    - Financial info
    - Location
    - Messages
    - Photos/videos
    - Files
    - Health
    - Contacts
    - App activity
    - Device IDs
```

## Policy Compliance

### Common Violations

**1. Ads Policy**
```
- No deceptive ads
- No inappropriate ads to children
- No ads that interfere with app
- Interstitials: allow dismissal
```

**2. User Data**
```
- Prominent disclosure for sensitive permissions
- Privacy policy required
- Handle data deletion requests
- Don't collect unnecessary data
```

**3. Deceptive Behavior**
```
- App must do what listing says
- No misleading claims
- No impersonation
- Accurate screenshots
```

**4. Malware**
```
- No code downloading
- No data harvesting
- Transparent functionality
```

**5. Families Policy**
```
If targeting children:
- No personalized ads
- Comply with COPPA
- Appropriate content only
```

### Permissions Best Practices

```xml
<!-- Only request what you need -->
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

<!-- Explain why you need them in-app -->
```

## Review Process

### Timeline
- Internal: Instant
- Closed/Open testing: Hours
- Production (new app): 1-3 days
- Production (update): Hours to 1 day

### Responding to Rejections
1. Read rejection email carefully
2. Check Policy Center for details
3. Make required changes
4. Reply via Appeals form if needed
5. Resubmit

## App Bundles & Dynamic Delivery

### Configuration APKs
Play Store generates optimized APKs for:
- Screen density
- CPU architecture
- Language

### On-Demand Modules
```kotlin
// build.gradle
dynamicFeatures = [':feature_camera']

// Download at runtime
val splitInstallManager = SplitInstallManagerFactory.create(context)
val request = SplitInstallRequest.newBuilder()
    .addModule("feature_camera")
    .build()
splitInstallManager.startInstall(request)
```

## Pre-Launch Report

Automatically run when you upload:
- Install and uninstall on real devices
- Basic UI tests (Robo)
- Crash detection
- Security scan
- Accessibility check

Review results before production release.

## Checklist

### Pre-Submission
- [ ] App fully tested
- [ ] No crashes in pre-launch report
- [ ] Privacy policy URL live
- [ ] All graphics prepared
- [ ] Descriptions written
- [ ] Content rating completed
- [ ] Data safety form done
- [ ] Target API level meets requirement

### Store Listing
- [ ] App name optimized
- [ ] Short description compelling
- [ ] Full description with keywords
- [ ] High-quality screenshots
- [ ] Feature graphic uploaded
- [ ] Contact details correct
- [ ] Category appropriate

### Release
- [ ] Signed AAB uploaded
- [ ] Version code incremented
- [ ] Release notes written
- [ ] Track selected
- [ ] Staged rollout configured (if production)

### Compliance
- [ ] Permissions minimized
- [ ] Ads policy followed
- [ ] Children policy (if applicable)
- [ ] Intellectual property respected
