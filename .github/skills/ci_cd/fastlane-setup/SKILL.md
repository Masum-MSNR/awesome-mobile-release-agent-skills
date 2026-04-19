---
name: fastlane-setup
description: Set up a complete Fastlane pipeline — Fastfile structure, lanes, match, pilot, supply. Use this when introducing Fastlane to an iOS or Android project.
---

# Fastlane Setup

## Instructions

Fastlane is the lingua franca of mobile release automation. This skill covers the **minimum viable Fastfile** for iOS + Android, set up so every lane runs identically on a developer laptop and on CI.

### 1. Install Fastlane

Pin the version in a `Gemfile` at the repo root:

```ruby
source "https://rubygems.org"
gem "fastlane", "~> 2.220"
plugins_path = File.join(File.dirname(__FILE__), "fastlane", "Pluginfile")
eval_gemfile(plugins_path) if File.exist?(plugins_path)
```

```bash
bundle config set --local path "vendor/bundle"
bundle install
```

Run as `bundle exec fastlane ...` everywhere.

### 2. Directory Layout

```text
ios/
  fastlane/
    Appfile
    Fastfile
    Matchfile
    Pluginfile
    metadata/
    screenshots/
android/
  fastlane/
    Appfile
    Fastfile
    Pluginfile
    metadata/
```

Separate `fastlane/` dirs per platform keep lanes tidy and allow platform-only lane runs from platform CI jobs.

### 3. `Appfile` (iOS)

```ruby
app_identifier("com.acme.app")
apple_id("ci@acme.com")
team_id("ABCDE12345")
itc_team_id("109876543")
```

### 4. `Appfile` (Android)

```ruby
json_key_file("")   # provided via env var PLAY_SERVICE_ACCOUNT_JSON_PATH
package_name("com.acme.app")
```

### 5. iOS `Fastfile`

```ruby
default_platform(:ios)

platform :ios do
  before_all { setup_ci }

  desc "Install certs & profiles via match"
  lane :certs do
    match(type: "development", readonly: true)
    match(type: "appstore", readonly: true)
  end

  desc "Build and upload to TestFlight"
  lane :beta do
    api_key = asc_api_key
    match(type: "appstore", readonly: true, api_key: api_key)
    build_app(
      workspace: "Runner.xcworkspace",
      scheme: "Runner",
      export_method: "app-store",
      export_options: "ExportOptions.plist",
    )
    upload_to_testflight(
      api_key: api_key,
      skip_waiting_for_build_processing: true,
      changelog: File.read("../CHANGELOG_CURRENT.txt"),
    )
    upload_symbols_to_crashlytics(gsp_path: "Runner/GoogleService-Info.plist")
  end

  desc "Submit latest TestFlight build to App Store Review"
  lane :release do
    deliver(
      api_key: asc_api_key,
      submit_for_review: true,
      automatic_release: false,
      phased_release: true,
      force: true,
      precheck_include_in_app_purchases: false,
    )
  end

  def asc_api_key
    app_store_connect_api_key(
      key_id: ENV["APP_STORE_CONNECT_KEY_ID"],
      issuer_id: ENV["APP_STORE_CONNECT_ISSUER_ID"],
      key_content: ENV["APP_STORE_CONNECT_KEY_CONTENT"],
      is_key_content_base64: true,
      duration: 1200,
    )
  end
end
```

### 6. Android `Fastfile`

```ruby
default_platform(:android)

platform :android do
  desc "Build AAB and upload to Play internal track"
  lane :internal do
    gradle(task: "bundleRelease",
           properties: {
             "versionName" => ENV["VERSION_NAME"],
             "versionCode" => ENV["GITHUB_RUN_NUMBER"],
           })
    upload_to_play_store(
      track: "internal",
      aab: "app/build/outputs/bundle/release/app-release.aab",
      json_key_data: ENV["PLAY_SERVICE_ACCOUNT_JSON"],
      release_status: "completed",
    )
  end

  desc "Promote internal to production at 5%"
  lane :promote_to_production do |options|
    upload_to_play_store(
      track: "internal",
      track_promote_to: "production",
      rollout: options[:rollout] || "0.05",
      skip_upload_aab: true,
      json_key_data: ENV["PLAY_SERVICE_ACCOUNT_JSON"],
    )
  end
end
```

### 7. `Matchfile`

```ruby
git_url("git@github.com:acme/certificates.git")
storage_mode("git")
type("appstore")
app_identifier(["com.acme.app"])
```

Encrypted with `MATCH_PASSWORD`. Store the password in CI secrets.

### 8. Environment Variables on CI

Minimum env set for a release run:

- `APP_STORE_CONNECT_KEY_ID`
- `APP_STORE_CONNECT_ISSUER_ID`
- `APP_STORE_CONNECT_KEY_CONTENT` (base64 `.p8`)
- `MATCH_PASSWORD`
- `MATCH_GIT_BASIC_AUTHORIZATION` or SSH deploy key
- `PLAY_SERVICE_ACCOUNT_JSON`
- `KEYSTORE_PATH` / `KEYSTORE_PASSWORD` / `KEY_ALIAS` / `KEY_PASSWORD`

### 9. Common Plugins

`ios/fastlane/Pluginfile`:

```ruby
gem "fastlane-plugin-versioning"
gem "fastlane-plugin-firebase_app_distribution"
gem "fastlane-plugin-sentry"
```

Install: `bundle exec fastlane add_plugin firebase_app_distribution`.

### 10. Running Locally vs CI

- Locally: `bundle exec fastlane ios beta` — uses your keychain, real Apple ID.
- CI: the same command — `setup_ci` creates a scoped temporary keychain; `match --readonly` never writes to the certs repo.

This parity is the main reason to use Fastlane. Do not introduce CI-only scripts that bypass lanes.

### 11. Anti-Patterns

- One giant Fastfile at the repo root mixing platforms.
- Using `--generate_apple_certs` in CI (can regenerate the distribution cert and break other teams).
- Hardcoding paths to build artefacts; use Fastlane variables (`SharedValues::IPA_OUTPUT_PATH`).
- Running `match` in `force` mode on CI. Always `readonly: true`.

### 12. Checklist
- [ ] `Gemfile` + `Gemfile.lock` committed; Fastlane pinned.
- [ ] Per-platform `fastlane/` directories with `Appfile`, `Fastfile`, `Matchfile`.
- [ ] ASC API key used, not Apple ID + password.
- [ ] `match readonly: true` in every CI lane.
- [ ] `setup_ci` called before signing on CI.
- [ ] Same lane names run locally and in CI, producing identical artefacts.
- [ ] Required env vars documented in `docs/release-env.md`.
