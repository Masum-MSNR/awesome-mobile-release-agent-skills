---
name: play-store-release
description: Release to Google Play using internal/closed/open tracks, AAB uploads, and the Data Safety form. Use this when shipping an Android build to the Play Console.
---

# Play Store Release

## Instructions

The Play Console has four public tracks: **internal**, **closed**, **open**, and **production**. A healthy release pipeline promotes the same AAB from internal → production without rebuilding.

### 1. Track Model

| Track | Audience | Max testers | Review |
|-------|----------|-------------|--------|
| Internal testing | Up to 100 testers by email | 100 | None (minutes) |
| Closed testing | Named tester lists or Google Groups | Unlimited | Same-day |
| Open testing | Anyone with the opt-in link | Unlimited | 1–7 days |
| Production | Public | — | 1–7 days (often 24h) |

Always upload to **internal** first. Promote the exact same AAB between tracks; never rebuild.

### 2. Build the AAB

```bash
# Flutter
flutter build appbundle --release \
  --build-name=1.4.2 --build-number=$GITHUB_RUN_NUMBER

# Native Gradle
./gradlew :app:bundleRelease \
  -PversionName=1.4.2 -PversionCode=$GITHUB_RUN_NUMBER
```

Output: `app/build/outputs/bundle/release/app-release.aab`.

### 3. Service Account for API Uploads

1. Play Console → Setup → API access → Create new service account.
2. Grant **Release manager** role for the app.
3. Download the JSON key; store as `PLAY_SERVICE_ACCOUNT_JSON` CI secret.

### 4. Upload with `fastlane supply`

`android/fastlane/Fastfile`:

```ruby
platform :android do
  desc "Upload AAB to the internal track"
  lane :internal do
    upload_to_play_store(
      track: "internal",
      aab: "../build/app/outputs/bundle/release/app-release.aab",
      json_key_data: ENV["PLAY_SERVICE_ACCOUNT_JSON"],
      release_status: "completed",
      skip_upload_metadata: false,
      skip_upload_images: true,
      skip_upload_screenshots: true,
    )
  end

  desc "Promote internal build to production at 5%"
  lane :promote do
    upload_to_play_store(
      track: "internal",
      track_promote_to: "production",
      rollout: "0.05",
      json_key_data: ENV["PLAY_SERVICE_ACCOUNT_JSON"],
      skip_upload_aab: true,
    )
  end
end
```

### 5. Release Notes

Write per-locale release notes under `android/fastlane/metadata/android/<locale>/changelogs/<versionCode>.txt`. `supply` picks them up automatically. Keep them ≤ 500 chars and user-facing.

### 6. Data Safety Form

Mandatory for every app. Declare:

- **Data collected** (categories: location, personal info, financial info, etc.).
- **Purpose** (analytics, fraud prevention, advertising).
- **Sharing** with third parties.
- **Encryption in transit** (true if you use HTTPS — you do).
- **Deletion request mechanism** (URL or in-app flow).

Update the form **before** uploading an AAB that changes data flows. See the dedicated `play-data-safety` skill for details.

### 7. Staged Rollout

Never jump to 100% on day one. Start at 5%, promote on evidence:

```bash
# Increase rollout
bundle exec fastlane supply --track production --rollout 0.20 --skip_upload_aab true

# Halt rollout (rollback in §10)
bundle exec fastlane supply --track production --rollout 0 --skip_upload_aab true
```

### 8. Testing Tracks With Real Users

- Open testing: create an opt-in URL, share in-app or via email to beta users.
- Closed testing: add Google Groups (`beta@acme.com`) in Play Console → Testing → Closed testing.
- Internal testing surface appears under Play Store → Menu → Manage apps → Beta.

### 9. In-App Updates

For fast pickup of critical updates, integrate [Play In-App Updates](https://developer.android.com/guide/playcore/in-app-updates):

```kotlin
val appUpdateManager = AppUpdateManagerFactory.create(context)
appUpdateManager.appUpdateInfo.addOnSuccessListener { info ->
    if (info.updateAvailability() == UpdateAvailability.UPDATE_AVAILABLE &&
        info.isUpdateTypeAllowed(AppUpdateType.IMMEDIATE)) {
        appUpdateManager.startUpdateFlowForResult(info, AppUpdateType.IMMEDIATE, activity, REQ_UPDATE)
    }
}
```

Use `IMMEDIATE` only for truly critical hotfixes; `FLEXIBLE` for everything else.

### 10. Halting / Rolling Back

- Halt staged rollout: set rollout to `0` (above). Users on the old version stay; new installs pause receiving the new one.
- Full rollback: re-promote the previous AAB from internal to production. You cannot decrement `versionCode`, so the "rollback" is really a new build of the previous source.

### 11. Review Pitfalls

- **Target API level**: Google enforces a rolling minimum (currently API 34 for new and updated apps). Bump promptly each August.
- **Permissions justification**: prominent disclosure screens are mandatory for sensitive permissions (location, SMS, call logs).
- **Ads SDKs**: declare in the app content section; ads for minors have extra rules.
- **Background location**: requires a separate review form.

### 12. Checklist
- [ ] AAB built (not APK) with a unique, monotonically increasing `versionCode`.
- [ ] Uploaded to **internal** track first; smoked tested on 2+ real devices.
- [ ] Data Safety form current and matches actual SDK behaviour.
- [ ] Release notes provided for every supported locale.
- [ ] Production rollout starts at **5%**, gated by the crash-free SLO.
- [ ] Service account JSON scoped to this app, stored only in CI secrets.
- [ ] Target API level meets Google's current requirement.
