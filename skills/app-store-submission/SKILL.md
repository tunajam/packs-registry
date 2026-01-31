# iOS App Store Submission

Complete guide to submitting apps to the Apple App Store.

## When to Apply

- Preparing first App Store submission
- Understanding review guidelines
- Creating App Store metadata
- Handling rejection reasons
- Managing app releases

## Prerequisites

- Apple Developer account ($99/year)
- App Store Connect access
- Valid distribution certificate
- App Store provisioning profile

## App Store Connect Setup

### Create App Record
1. Go to App Store Connect
2. My Apps > + (New App)
3. Fill in:
   - Platform: iOS
   - Name: App name (unique on store)
   - Primary language
   - Bundle ID (from Xcode)
   - SKU (internal reference)

### Required Metadata

```yaml
App Information:
  - Name: 30 characters max
  - Subtitle: 30 characters max
  - Privacy Policy URL: Required
  - Category: Primary + Secondary
  - Content Rights: If using third-party content

Version Information:
  - Description: 4000 characters max
  - Keywords: 100 characters total (comma-separated)
  - Support URL: Required
  - Marketing URL: Optional
  - What's New: Required for updates

Pricing & Availability:
  - Price: Free or select tier
  - Availability: Countries/regions
  - Pre-Order: Optional
```

## Screenshots Requirements

### Sizes Required
```
iPhone 6.9" (1320 x 2868 or 2868 x 1320)
  - iPhone 15 Pro Max, 16 Pro Max
  
iPhone 6.7" (1290 x 2796 or 2796 x 1290)
  - iPhone 15 Plus, 15 Pro Max, 14 Plus, 14 Pro Max
  
iPhone 6.5" (1242 x 2688 or 2688 x 1242)
  - iPhone 11 Pro Max, XS Max

iPhone 5.5" (1242 x 2208 or 2208 x 1242)
  - iPhone 8 Plus, 7 Plus, 6s Plus

iPad Pro 12.9" (2048 x 2732)
  - 3rd gen and later

iPad Pro 11" (1668 x 2388)
  - Required if supporting iPad
```

### Best Practices
- 3-10 screenshots per device
- Show key features
- Use device frames (optional)
- Localize for each language
- First screenshot is most important

## App Preview Videos

```yaml
Requirements:
  - Duration: 15-30 seconds
  - Format: H.264, 30fps
  - Audio: AAC, stereo
  - Resolution: Match screenshot sizes
  - No device frames in video
  
Tips:
  - Show actual app usage
  - No iPhone hardware shown
  - Can include on-screen text
  - Must represent actual app
```

## Build Submission

### Using Xcode
```bash
# Archive
Product > Archive

# Upload
Window > Organizer > Distribute App
  > App Store Connect > Upload
```

### Using EAS (Expo)
```bash
# Build for submission
eas build --platform ios --profile production

# Submit to App Store
eas submit --platform ios
```

### Using Fastlane
```ruby
# Fastfile
lane :release do
  build_app(
    scheme: "MyApp",
    export_method: "app-store"
  )
  upload_to_app_store(
    skip_screenshots: true,
    skip_metadata: true
  )
end
```

## Review Guidelines Summary

### Common Rejection Reasons

**1. Bugs and Crashes**
- Test thoroughly before submitting
- Check all user flows
- Test on multiple devices

**2. Broken Links**
- Verify all URLs work
- Privacy policy must be accessible
- Support links must function

**3. Placeholder Content**
- Remove "lorem ipsum"
- Replace sample data
- Complete all app sections

**4. Incomplete Information**
- Fill all required metadata
- Explain features that need demo accounts
- Provide test credentials if needed

**5. Login Issues**
```
Required for apps with accounts:
- Demo account in review notes
- "Sign in with Apple" if social logins exist
- Guest mode option (recommended)
```

**6. Privacy Issues**
```yaml
Required:
  - Privacy policy URL
  - App Privacy details (nutrition labels)
  - Purpose strings for permissions:
    - NSCameraUsageDescription
    - NSPhotoLibraryUsageDescription
    - NSLocationWhenInUseUsageDescription
    - etc.
```

### Guidelines to Know

```yaml
Safety:
  - No objectionable content
  - User-generated content needs moderation
  - Apps must be stable

Performance:
  - Must be complete app
  - No beta/demo labels
  - Hardware compatibility accurate

Business:
  - In-app purchases for digital goods
  - No external payment links for digital content
  - Physical goods can use other payment methods

Design:
  - Follow Human Interface Guidelines
  - No copycat apps
  - Minimum functionality required

Legal:
  - Respect intellectual property
  - GDPR compliance if in EU
  - Children's privacy (COPPA) if applicable
```

## App Store Review Notes

```
Format for review notes:

Demo Account:
Email: demo@example.com
Password: TestPassword123

Special Instructions:
1. To test feature X, tap on "Settings"
2. The in-app purchase is in sandbox mode
3. Push notifications require device (not simulator)

Device Requirements:
- Best experienced on iPhone 15+
- Requires camera for feature Y
```

## Post-Submission

### Review Times
- Standard: 24-48 hours typical
- Expedited: Request for critical fixes

### Responding to Rejections
1. Read rejection reason carefully
2. Fix all mentioned issues
3. Reply via Resolution Center
4. Resubmit with notes explaining changes

### Phased Release
```yaml
Options:
  - Manual release: You control when
  - Automatic after approval
  - Phased release: 1-7 days rollout
    - Day 1: 1%
    - Day 2: 2%
    - Day 3: 5%
    - Day 4: 10%
    - Day 5: 20%
    - Day 6: 50%
    - Day 7: 100%
```

## Checklist

### Pre-Submission
- [ ] All features complete and tested
- [ ] No crashes or critical bugs
- [ ] All placeholder content removed
- [ ] Privacy policy URL working
- [ ] All required screenshots
- [ ] App preview video (if any)
- [ ] Keywords optimized
- [ ] Description complete
- [ ] Test on multiple devices
- [ ] Demo account ready

### Metadata
- [ ] App name unique
- [ ] Subtitle compelling
- [ ] Description SEO-optimized
- [ ] Keywords research done
- [ ] Category appropriate
- [ ] Age rating accurate
- [ ] App Privacy completed

### Technical
- [ ] Bundle ID correct
- [ ] Version number incremented
- [ ] Build number unique
- [ ] Icons all sizes provided
- [ ] Launch screen configured
- [ ] Permission strings clear
