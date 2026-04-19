---
name: bitrise-codemagic
description: Choose between Bitrise and Codemagic — stack choice, workflows, caches, and secrets. Use this when you need managed macOS runners without maintaining your own.
---

# Bitrise & Codemagic

## Instructions

Bitrise and Codemagic are managed mobile CI platforms with first-class iOS + Android support, hosted macOS runners, and built-in signing integration. Use them when the team does not want to run self-hosted macs or pay GitHub Actions macOS minutes at scale.

### 1. When to Pick Each

| Platform | Strengths | Pick when |
|----------|-----------|-----------|
| Bitrise | Large step library, strong Android support, flexible workflows | You want a marketplace of reusable steps |
| Codemagic | First-class Flutter & React Native templates, M2-class macs | Cross-platform framework projects, GPU builds |

Both support webhooks from GitHub/GitLab/Bitbucket, artefact storage, and store submission.

### 2. Stack Choice

- **macOS image:** pin to a specific Xcode version (e.g. `Xcode 15.4`). Drift between major Xcode versions breaks builds silently.
- **Linux image (Android):** Ubuntu 22.04 LTS with AGP-compatible Java (JDK 17 for AGP 8.x).
- **Concurrency:** iOS needs 1 dedicated mac per build; Android can share Linux runners.

### 3. Bitrise Workflow (`bitrise.yml`)

```yaml
format_version: "13"
default_step_lib_source: https://github.com/bitrise-io/bitrise-steplib.git
project_type: other

app:
  envs:
    - FASTLANE_XCODE_LIST_TIMEOUT: "120"

trigger_map:
  - push_branch: main
    workflow: ios-beta
  - tag: "v*.*.*"
    workflow: ios-release

workflows:
  ios-beta:
    steps:
      - activate-ssh-key@4: {}
      - git-clone@8: {}
      - cache-pull@2: {}
      - script@1:
          inputs:
            - content: |
                bundle config set path vendor/bundle
                bundle install
      - cocoapods-install@2: {}
      - fastlane@3:
          inputs:
            - work_dir: ios
            - lane: beta
      - deploy-to-bitrise-io@2: {}
      - cache-push@2:
          inputs:
            - cache_paths: |
                $HOME/.gem
                $BITRISE_SOURCE_DIR/ios/Pods
                $BITRISE_SOURCE_DIR/vendor/bundle
```

### 4. Codemagic Workflow (`codemagic.yaml`)

```yaml
workflows:
  ios-beta:
    name: iOS beta
    instance_type: mac_mini_m2
    max_build_duration: 60
    environment:
      xcode: 15.4
      cocoapods: default
      ruby: 3.2.2
      groups:
        - appstore_credentials
        - match_credentials
      vars:
        LANG: en_US.UTF-8
    triggering:
      events: [push]
      branch_patterns:
        - pattern: main
          include: true
    scripts:
      - name: Install gems
        script: bundle config set path vendor/bundle && bundle install
      - name: Install pods
        script: cd ios && pod install --repo-update
      - name: Fastlane beta
        script: cd ios && bundle exec fastlane beta
    artifacts:
      - build/ios/ipa/*.ipa
      - /tmp/xcodebuild_logs/*.log

  android-internal:
    name: Android internal
    instance_type: linux_x2
    environment:
      groups:
        - play_credentials
        - android_signing
      java: 17
    triggering:
      events: [push]
      branch_patterns:
        - pattern: main
          include: true
    scripts:
      - name: Decode keystore
        script: echo $ANDROID_KEYSTORE_BASE64 | base64 -d > android/upload-keystore.jks
      - name: Fastlane internal
        script: cd android && bundle exec fastlane internal
    artifacts:
      - android/app/build/outputs/bundle/release/*.aab
```

### 5. Caching

Cache these on every build:

- `~/.gem` / `vendor/bundle` (Ruby)
- `~/.gradle/caches` (Android)
- `ios/Pods` + `~/Library/Caches/CocoaPods` (iOS)
- `~/.pub-cache` (Flutter / Dart)
- `node_modules` (React Native)

Key caches by the hash of the corresponding lockfile: `Gemfile.lock`, `Podfile.lock`, `gradle/wrapper/gradle-wrapper.properties`, `pubspec.lock`.

### 6. Secrets Management

Both platforms store encrypted env vars grouped logically:

- `appstore_credentials`: `APP_STORE_CONNECT_KEY_ID`, `_ISSUER_ID`, `_KEY_CONTENT`
- `match_credentials`: `MATCH_PASSWORD`, `MATCH_GIT_BASIC_AUTHORIZATION`
- `play_credentials`: `PLAY_SERVICE_ACCOUNT_JSON` (file or string)
- `android_signing`: `ANDROID_KEYSTORE_BASE64`, `KEYSTORE_PASSWORD`, `KEY_ALIAS`, `KEY_PASSWORD`

Scope groups to specific workflows so PR builds never see release secrets.

### 7. Signing Integration

Codemagic and Bitrise both offer native Apple certificate management (automatic signing with their UI-uploaded certs). **Do not use it** — stick with `match` so developer laptops, CI, and future migrations stay identical.

### 8. Artefact Storage & Notifications

- Bitrise: `deploy-to-bitrise-io` step; artefacts retained per plan.
- Codemagic: `artifacts:` list; expirable URLs emailed on success.

Notify on failure via Slack step:

```yaml
# Bitrise
- slack@4:
    inputs:
      - webhook_url: $SLACK_WEBHOOK
      - channel: "#release"
      - message: "Build $BITRISE_BUILD_URL failed"
    run_if: .IsBuildFailed
```

### 9. Cost Control

- Run PR builds on Linux/Ubuntu where possible; restrict macOS to iOS-required jobs.
- Use path filters to skip mobile CI for docs-only PRs.
- Fail fast: analyse/lint first, build last.

### 10. Migration Tips

Moving to/from GitHub Actions:

- Keep Fastlane as the single source of truth; CI is a wrapper.
- Export secrets explicitly; each platform has a different env var injection format.
- Pin toolchains (`Xcode`, `JDK`, `Ruby`) identically to minimise drift.

### 11. Anti-Patterns

- Defining signing steps in the CI UI only — unreproducible locally.
- Letting the CI platform auto-update Xcode versions; always pin.
- Sharing one giant "env group" across PR and release workflows.
- Skipping cache — macOS minutes are the single largest mobile CI cost.

### 12. Checklist
- [ ] Platform chosen with a documented rationale.
- [ ] Xcode and JDK versions pinned in workflow YAML.
- [ ] Caches configured for Ruby, Gradle, CocoaPods, and Pub.
- [ ] Secret groups scoped per workflow; PR builds isolated from release secrets.
- [ ] Fastlane lanes are the actual build commands; CI YAML is a thin wrapper.
- [ ] Slack / email notifications configured for failed release builds.
- [ ] Cost monitored monthly; macOS minutes budgeted.
