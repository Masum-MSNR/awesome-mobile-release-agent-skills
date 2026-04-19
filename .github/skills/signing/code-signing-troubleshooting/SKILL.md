---
name: code-signing-troubleshooting
description: Diagnose and fix common Apple and Google code-signing errors. Use this when a release build fails to sign, upload, or install.
---

# Code Signing Troubleshooting

## Instructions

Signing errors are almost always one of five categories: **wrong certificate, wrong profile, wrong entitlements, wrong bundle id, or a stale cache**. Work through this skill top-to-bottom — the order matches how often each fault occurs in practice.

### 1. First, Get the Full Error

Xcode and Gradle both truncate signing errors. Re-run with verbose flags:

```bash
# iOS
xcodebuild -workspace Runner.xcworkspace -scheme Runner \
  -configuration Release archive -archivePath out.xcarchive \
  CODE_SIGN_STYLE=Manual -verbose | tee build.log

# Android
./gradlew :app:bundleRelease --info --stacktrace
```

Grep for `error:`, `Signing Identity`, and `Provisioning Profile`.

### 2. Apple — `No signing certificate "iOS Distribution" found`

Cause: no matching certificate in the keychain.

Fix:

```bash
security find-identity -v -p codesigning
# Expect at least one "Apple Distribution: <Team>" line.
bundle exec fastlane match appstore --readonly
```

If `match` reports a cert exists remotely but locally missing, delete `~/Library/MobileDevice/Provisioning Profiles` and re-run.

### 3. Apple — `Provisioning profile doesn't include the entitlement`

Cause: entitlements file references a capability not enabled on the App ID.

Fix:

1. Developer portal → Identifiers → select App ID → enable the capability.
2. Regenerate profile: `bundle exec fastlane match nuke appstore && bundle exec fastlane match appstore`.
3. Confirm `Runner.entitlements` matches what the portal shows.

### 4. Apple — `The executable was signed with invalid entitlements`

Upload to App Store Connect rejected. Usually `get-task-allow = true` leaked into a release build (debug entitlements in a release build). In `ExportOptions.plist`:

```xml
<key>signingStyle</key>
<string>manual</string>
<key>method</key>
<string>app-store</string>
```

Never set `-allowProvisioningUpdates` on a release export; it can pull a development profile.

### 5. Apple — `App Store Connect reported an error uploading`

Often transient. Retry with the API key, not password:

```bash
xcrun altool --upload-app -f MyApp.ipa \
  --apiKey "$APP_STORE_CONNECT_KEY_ID" \
  --apiIssuer "$APP_STORE_CONNECT_ISSUER_ID"
```

If it persists, check [Apple System Status](https://developer.apple.com/system-status/) before assuming a local fault.

### 6. Google — `Your APK or Android App Bundle was signed with a key that is not in Play Console`

Cause: uploading with a new keystore.

Fix: revert to the original upload key. If lost, Play Console → App integrity → Request upload key reset (wait 1–2 days). Do not rename bundles to work around this — it creates a duplicate listing.

### 7. Google — `Signature verification failed: v1 signature scheme missing`

Cause: `minSdk < 24` but v2+ only. Fix in `app/build.gradle.kts`:

```kotlin
android.signingConfigs.getByName("release") {
    enableV1Signing = true
    enableV2Signing = true
    enableV3Signing = true
}
```

### 8. Google — `INSTALL_PARSE_FAILED_NO_CERTIFICATES` on device

The APK is unsigned or signature corrupted. Re-sign:

```bash
apksigner sign --ks upload-keystore.jks \
  --ks-key-alias upload \
  --ks-pass env:KEYSTORE_PASSWORD \
  --out app-release-signed.apk \
  app-release-unsigned.apk

apksigner verify --verbose app-release-signed.apk
```

### 9. Stale Caches (Both Platforms)

When nothing else explains it, clear:

```bash
# iOS
rm -rf ~/Library/Developer/Xcode/DerivedData
rm -rf ~/Library/MobileDevice/Provisioning\ Profiles/*.mobileprovision

# Android
./gradlew clean
rm -rf ~/.gradle/caches/transforms-*
```

Re-run the signing lane from scratch.

### 10. Diagnostic Commands Cheat Sheet

```bash
# Inspect an IPA's embedded profile
unzip -p MyApp.ipa "Payload/*.app/embedded.mobileprovision" \
  | security cms -D | plutil -p -

# Inspect an AAB
bundletool dump manifest --bundle=app-release.aab
bundletool dump resources --bundle=app-release.aab

# List signing identities
security find-identity -v -p codesigning

# Check APK signatures
apksigner verify --print-certs app-release.apk
```

### 11. Escalation Path

1. Reproduce locally with the exact CI command.
2. Compare the failing artefact against the last successful one (`diff` the profile plists).
3. Check cert/profile expiry in the Developer portal and Play Console.
4. Open an Apple Developer Technical Support ticket or Play Console help request only after the above three steps.

### 12. Checklist
- [ ] Full verbose log captured before touching anything.
- [ ] Cert/profile expiry confirmed current.
- [ ] Entitlements file matches App ID capabilities.
- [ ] `match` or `supply` readonly mode used on CI to prevent accidental regeneration.
- [ ] Derived Data / Gradle caches cleared on "nothing changed but it broke" issues.
- [ ] Fix written up in `docs/signing-playbook.md` for the next time.
