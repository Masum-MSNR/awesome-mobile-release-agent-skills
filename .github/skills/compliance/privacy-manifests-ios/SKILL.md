---
name: privacy-manifests-ios
description: Declare iOS Required Reason APIs and tracking in PrivacyInfo.xcprivacy. Use this when adding SDKs, new APIs, or preparing for App Store submission.
---

# iOS Privacy Manifests

## Instructions

Since May 2024, Apple requires every app and third-party SDK to ship a `PrivacyInfo.xcprivacy` manifest declaring data collection, tracking domains, and use of **Required Reason APIs**. Missing declarations cause App Store rejection.

### 1. What the Manifest Contains

Four top-level keys:

- `NSPrivacyTracking` (`Bool`) — does the app/SDK track users?
- `NSPrivacyTrackingDomains` (`Array<String>`) — domains used only after ATT consent.
- `NSPrivacyCollectedDataTypes` — categories of collected data and purposes.
- `NSPrivacyAccessedAPITypes` — Required Reason APIs used, with approved reason codes.

### 2. File Location

- Swift package: at the root of the package.
- Xcode project: `Runner/PrivacyInfo.xcprivacy` added to the target.
- CocoaPods: pod maintainer ships it inside the `.podspec` resources.

### 3. Minimal Template

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>NSPrivacyTracking</key>
  <false/>
  <key>NSPrivacyTrackingDomains</key>
  <array/>
  <key>NSPrivacyCollectedDataTypes</key>
  <array/>
  <key>NSPrivacyAccessedAPITypes</key>
  <array>
    <dict>
      <key>NSPrivacyAccessedAPIType</key>
      <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
      <key>NSPrivacyAccessedAPITypeReasons</key>
      <array><string>CA92.1</string></array>
    </dict>
  </array>
</dict>
</plist>
```

### 4. Required Reason API Categories

The five Apple-designated categories, each with approved reason codes:

| Category | Typical use | Example reason code |
|----------|-------------|---------------------|
| `FileTimestamp` | File modification dates | `C617.1` |
| `SystemBootTime` | Measure elapsed time across launches | `35F9.1` |
| `DiskSpace` | Check free space before write | `85F4.1` |
| `ActiveKeyboards` | Keyboard extensions only | `3EH0.1` |
| `UserDefaults` | Persist small settings | `CA92.1`, `1C8F.1` |

Full list: [Apple — Required Reasons APIs](https://developer.apple.com/documentation/bundleresources/privacy_manifest_files/describing_use_of_required_reason_api).

### 5. Declaring Collected Data

```xml
<key>NSPrivacyCollectedDataTypes</key>
<array>
  <dict>
    <key>NSPrivacyCollectedDataType</key>
    <string>NSPrivacyCollectedDataTypeEmailAddress</string>
    <key>NSPrivacyCollectedDataTypeLinked</key>
    <true/>
    <key>NSPrivacyCollectedDataTypeTracking</key>
    <false/>
    <key>NSPrivacyCollectedDataTypePurposes</key>
    <array>
      <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
    </array>
  </dict>
</array>
```

Must match the App Privacy nutrition labels in App Store Connect.

### 6. Declaring Tracking Domains

If your app tracks users (ATT-gated), list the domains:

```xml
<key>NSPrivacyTracking</key><true/>
<key>NSPrivacyTrackingDomains</key>
<array>
  <string>analytics.thirdparty.com</string>
  <string>ads.network.example</string>
</array>
```

If ATT is denied, the OS blocks traffic to these domains automatically.

### 7. Third-Party SDKs — Commonly Required

Apple maintains a [list of SDKs](https://developer.apple.com/support/third-party-SDK-requirements) that must ship a privacy manifest AND be signed. As of 2024–25, covered SDKs include Firebase, GoogleSignIn, Adjust, Branch, Amplitude, RevenueCat, and many others. You cannot submit if any of these is embedded without its manifest + signature.

### 8. Auditing Your Build

```bash
# Find all manifests inside a .xcarchive
find MyApp.xcarchive -name "PrivacyInfo.xcprivacy" -print

# Dump a specific manifest
plutil -p Runner/PrivacyInfo.xcprivacy

# Check SDK signature
codesign -dv --verbose=4 Pods/FBSDKCoreKit/*.framework
```

Xcode 15.3+ emits a **Privacy Report** when you archive:

```
Product → Archive → Archive → Distribute App → App Store Connect
→ Upload → the privacy report tab
```

Review and export the PDF before submission.

### 9. Common Errors

- **`ITMS-91053: Missing API declaration`** — An API from a Required Reason category is used without a reason code. Grep for the API (e.g. `CreationDate`, `ModificationDate`) and add the matching reason.
- **`ITMS-91056: Invalid privacy manifest`** — Typos in keys. Run `plutil -lint PrivacyInfo.xcprivacy`.
- **`ITMS-91065: Missing signature`** — A covered SDK is not signed. Update to a version ≥ the minimum Apple requires.

### 10. CI Lint

```bash
#!/usr/bin/env bash
# scripts/check-privacy-manifests.sh
set -e
missing=0
for sdk in $(ls Pods | grep -v '^Headers$\|^Local Podspecs$\|^Target Support Files$'); do
  if grep -q "^$sdk\$" scripts/privacy-manifest-required.txt; then
    if ! find "Pods/$sdk" -name 'PrivacyInfo.xcprivacy' | grep -q .; then
      echo "Missing PrivacyInfo.xcprivacy in $sdk"
      missing=$((missing+1))
    fi
  fi
done
exit $missing
```

Run in the PR pipeline so missing manifests block merge.

### 11. Keeping It in Sync

Every new SDK PR must include:

- Bump in `Podfile` / SwiftPM manifest.
- Verification the pod includes `PrivacyInfo.xcprivacy`.
- Update to the app's own manifest if the SDK introduces new data categories.
- Update to App Store Connect App Privacy labels if categories changed.

### 12. Checklist
- [ ] `Runner/PrivacyInfo.xcprivacy` added to the target; schema validated with `plutil -lint`.
- [ ] Every Required Reason API used has an approved reason code.
- [ ] `NSPrivacyTracking` and `NSPrivacyTrackingDomains` accurate for current SDKs.
- [ ] Third-party SDKs updated to signed, manifest-bearing versions.
- [ ] CI lint fails on missing manifests in covered SDKs.
- [ ] Privacy Report reviewed and exported at archive time.
- [ ] App Store Connect App Privacy labels match the manifest.
