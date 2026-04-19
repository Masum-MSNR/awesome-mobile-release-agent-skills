---
name: ios-signing-provisioning
description: Manage Apple certificates, provisioning profiles, automatic vs manual signing, and fastlane match. Use this when setting up iOS release signing.
---

# iOS Signing & Provisioning

## Instructions

Apple's signing model has three moving parts: **certificates** (who you are), **identifiers** (what your app is), and **provisioning profiles** (what the app is allowed to do, on which devices). Automate all three with `fastlane match`.

### 1. Certificate Types

| Certificate | Purpose |
|-------------|---------|
| Apple Development | Run on personal devices; Xcode Previews |
| Apple Distribution | TestFlight + App Store + Ad Hoc + Enterprise |
| Developer ID | Notarized macOS apps outside the App Store |
| Pass Type ID | Wallet passes |

For most apps you only need **Apple Development** + **Apple Distribution**.

### 2. `fastlane match` Setup

`match` stores encrypted certs + profiles in a private Git repo (or S3/GCS). Every teammate and every CI runner pulls the same artefacts.

```bash
bundle exec fastlane match init
# choose git, enter repo URL e.g. git@github.com:acme/certificates.git
```

`ios/fastlane/Matchfile`:

```ruby
git_url("git@github.com:acme/certificates.git")
storage_mode("git")
type("development")
app_identifier(["com.acme.app", "com.acme.app.notifications"])
username("ci@acme.com")
```

Then:

```bash
bundle exec fastlane match development
bundle exec fastlane match appstore
bundle exec fastlane match adhoc       # only if you ship Ad Hoc builds
```

### 3. Automatic vs Manual Signing

**Automatic signing** (`ProvisioningStyle = Automatic`) lets Xcode pick profiles. Good for local development, dangerous for CI because Xcode will silently regenerate profiles and invalidate cached certificates.

**Manual signing** (the default for CI) pins `PROVISIONING_PROFILE_SPECIFIER` and `CODE_SIGN_IDENTITY` to the names `match` installs. In `ios/Runner.xcodeproj/project.pbxproj`:

```text
CODE_SIGN_STYLE = Manual;
CODE_SIGN_IDENTITY = "Apple Distribution";
PROVISIONING_PROFILE_SPECIFIER = "match AppStore com.acme.app";
DEVELOPMENT_TEAM = ABCDE12345;
```

### 4. App Store Connect API Key

Prefer API keys over password-based login:

1. App Store Connect → Users and Access → Keys → Generate API Key (role: App Manager).
2. Download the `.p8` (one-time).
3. Store `APP_STORE_CONNECT_KEY_ID`, `APP_STORE_CONNECT_ISSUER_ID`, and the base64-encoded `.p8` in CI secrets.

`ios/fastlane/Fastfile`:

```ruby
lane :beta do
  api_key = app_store_connect_api_key(
    key_id: ENV["APP_STORE_CONNECT_KEY_ID"],
    issuer_id: ENV["APP_STORE_CONNECT_ISSUER_ID"],
    key_content: ENV["APP_STORE_CONNECT_KEY_CONTENT"],
    is_key_content_base64: true,
    duration: 1200,
  )
  setup_ci
  match(type: "appstore", readonly: true, api_key: api_key)
  build_app(workspace: "Runner.xcworkspace", scheme: "Runner")
  upload_to_testflight(api_key: api_key, skip_waiting_for_build_processing: true)
end
```

### 5. `setup_ci` and the Temporary Keychain

`setup_ci` creates a scoped keychain on CI runners, avoids the interactive password prompt, and makes `match` idempotent. Always include it at the top of CI-bound lanes.

### 6. Provisioning Profile Entitlements

Entitlements in `Runner.entitlements` must match the capabilities enabled on the App ID in the Developer portal. Mismatches produce `Provisioning profile doesn't include the entitlement` errors. Common entitlements:

- `aps-environment` (push)
- `com.apple.developer.associated-domains` (Universal Links)
- `com.apple.developer.in-app-payments` (Apple Pay)
- `com.apple.developer.networking.wifi-info`

Toggle capabilities in the portal, then re-run `match` to regenerate profiles.

### 7. Rotating Certificates

Distribution certs expire in 1 year. A day before expiry:

```bash
bundle exec fastlane match nuke distribution   # revoke old certs (careful!)
bundle exec fastlane match appstore            # regenerate and push to Git
```

`match nuke` revokes **all** certs of that type across the team — coordinate in Slack before running.

### 8. Verifying a Signed IPA

```bash
unzip -p MyApp.ipa "Payload/*.app/embedded.mobileprovision" \
  | security cms -D \
  | plutil -p - | grep -E 'Name|TeamName|ExpirationDate'
codesign -dv --verbose=4 Payload/MyApp.app
```

### 9. Common Gotchas

- **Bundle identifier mismatch** between `Info.plist`, App ID, and profile name.
- **Provisioning profile expired** (profiles last 1 year; certs 1 year).
- **Multiple distribution certs** on the account — `match nuke` or delete extras in the portal; Apple limits you to 2 distribution certs per account.
- **Enterprise certs** require a separate agreement and cannot be published to App Store.

### 10. Checklist
- [ ] `fastlane match` initialised with a private, encrypted Git repo.
- [ ] Xcode project uses **manual** signing on release configuration.
- [ ] App Store Connect API key (`.p8`) stored in CI secrets; no password login.
- [ ] `setup_ci` called in every CI lane before `match` and `build_app`.
- [ ] Entitlements in `.entitlements` match App ID capabilities.
- [ ] Certificate expiry dates tracked in the team calendar.
