---
name: app-store-release
description: Release to the App Store using App Store Connect, TestFlight, metadata, and screenshots. Use this when shipping an iOS build to Apple.
---

# App Store Release

## Instructions

Apple's release path is: **Archive → Upload → Process → TestFlight → Submit for Review → Phased Release**. Automate each step with `pilot` and `deliver`; never click through the Connect UI on release day.

### 1. Build the IPA

```bash
# Flutter
flutter build ipa --release \
  --export-options-plist=ios/ExportOptions.plist \
  --build-name=1.4.2 --build-number=$GITHUB_RUN_NUMBER

# Native
xcodebuild -workspace MyApp.xcworkspace -scheme MyApp \
  -configuration Release -archivePath build/MyApp.xcarchive archive
xcodebuild -exportArchive -archivePath build/MyApp.xcarchive \
  -exportPath build/ipa -exportOptionsPlist ios/ExportOptions.plist
```

Minimal `ExportOptions.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key><string>app-store</string>
  <key>signingStyle</key><string>manual</string>
  <key>teamID</key><string>ABCDE12345</string>
  <key>uploadBitcode</key><false/>
  <key>uploadSymbols</key><true/>
  <key>provisioningProfiles</key>
  <dict>
    <key>com.acme.app</key>
    <string>match AppStore com.acme.app</string>
  </dict>
</dict>
</plist>
```

### 2. Upload with `pilot` (TestFlight)

```ruby
lane :beta do
  api_key = app_store_connect_api_key(
    key_id: ENV["APP_STORE_CONNECT_KEY_ID"],
    issuer_id: ENV["APP_STORE_CONNECT_ISSUER_ID"],
    key_content: ENV["APP_STORE_CONNECT_KEY_CONTENT"],
    is_key_content_base64: true,
  )
  setup_ci
  match(type: "appstore", readonly: true, api_key: api_key)
  build_app(workspace: "MyApp.xcworkspace", scheme: "MyApp",
            export_options: "ios/ExportOptions.plist")
  upload_to_testflight(
    api_key: api_key,
    skip_waiting_for_build_processing: true,
    changelog: File.read("../CHANGELOG_CURRENT.txt"),
    distribute_external: true,
    groups: ["External Beta"],
  )
end
```

Alternative: `xcrun altool --upload-app -f MyApp.ipa --apiKey ... --apiIssuer ...`.

### 3. TestFlight Groups

- **Internal testers** (App Store Connect users with roles) — no review, immediate access, up to 100.
- **External testers** — require Beta App Review (first build only, minor updates often auto-approved). Up to 10,000.

Automate group membership with `pilot`:

```bash
bundle exec fastlane pilot add user@example.com --groups "External Beta"
```

### 4. Metadata with `deliver`

`ios/fastlane/metadata/en-US/` tree:

```text
metadata/
  en-US/
    name.txt                 # ≤ 30 chars
    subtitle.txt             # ≤ 30 chars
    description.txt          # ≤ 4000 chars
    keywords.txt             # comma-separated, ≤ 100 chars
    release_notes.txt        # ≤ 4000 chars, this version
    support_url.txt
    marketing_url.txt
    privacy_url.txt
```

Submit:

```ruby
lane :release do
  api_key = app_store_connect_api_key(...)
  deliver(
    api_key: api_key,
    submit_for_review: true,
    automatic_release: false,
    phased_release: true,
    force: true,               # skip HTML preview
    precheck_include_in_app_purchases: false,
    submission_information: {
      add_id_info_uses_idfa: false,
      export_compliance_uses_encryption: false,
    },
  )
end
```

### 5. Screenshots

Required sizes (as of Xcode 15): 6.7" iPhone, 6.1" iPhone (optional fallback), 12.9" iPad Pro, 13" iPad Pro. Generate with `fastlane snapshot`:

```bash
bundle exec fastlane snapshot init
# Configure Snapfile with devices + languages
bundle exec fastlane snapshot
```

Place results under `ios/fastlane/screenshots/<locale>/`.

### 6. Export Compliance

Apps using only HTTPS qualify for the simplified compliance path. Set in `Info.plist` to skip the per-build question:

```xml
<key>ITSAppUsesNonExemptEncryption</key>
<false/>
```

If you use custom crypto, you need annual self-classification and possibly an ERN/CCATS.

### 7. Phased Release

Enable via `deliver(phased_release: true)`. Apple's 7-day ramp delivers updates to 1%, 2%, 5%, 10%, 20%, 50%, 100% of devices set to auto-update. See the dedicated `phased-release-ios` skill for pausing/resuming.

### 8. Build Processing Gotchas

- Processing can take 15–60 minutes; plan accordingly.
- `missing_compliance` email on first upload of a version — open the build in TestFlight and answer export compliance.
- **Invalid Bundle**: missing required device capabilities, unsupported architectures, or Info.plist missing `CFBundleIconName`. Check the email details.
- dSYMs: upload with `upload_symbols_to_crashlytics` or `sentry-cli debug-files upload` right after `build_app`.

### 9. Expedited Review

For critical bugs affecting users, request expedited review (App Store Connect → Contact Us → App Review → Request Expedited Review). Use sparingly; Apple tracks abuse.

### 10. App Privacy "Nutrition Labels"

Separate from `PrivacyInfo.xcprivacy`. Fill under App Store Connect → App Privacy. Must match actual data practices across the app **and** all embedded SDKs (ads, analytics, crash reporters).

### 11. Checklist
- [ ] IPA built with manual signing and `match`-managed profiles.
- [ ] Uploaded via `pilot` with App Store Connect API key; no password login.
- [ ] TestFlight external group distribution enabled; changelog attached.
- [ ] Metadata, screenshots, and privacy URL kept in git under `ios/fastlane/metadata`.
- [ ] `phased_release: true` on production submissions.
- [ ] dSYMs uploaded to the crash reporter.
- [ ] App Privacy labels match actual SDK behaviour.
