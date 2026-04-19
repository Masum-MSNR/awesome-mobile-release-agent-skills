---
name: github-actions-mobile-release
description: Build and release mobile apps from GitHub Actions — macOS runners, AAB/IPA artifacts, signing secrets, and tag-triggered deploys. Use this when GitHub Actions is the CI of record.
---

# GitHub Actions for Mobile Release

## Instructions

GitHub Actions is the default CI for many mobile teams because the workflows live next to the code. This skill gives you a production-ready setup for iOS + Android with proper secret handling, caching, and tag-triggered releases.

### 1. Workflow Layout

```text
.github/workflows/
  pr.yml                # lint + test on every PR
  main.yml              # internal/TestFlight on every merge to main
  release.yml           # production release on v*.*.* tags
```

Keep each file single-purpose; avoid one mega-workflow with 20 jobs.

### 2. PR Workflow

```yaml
name: PR
on:
  pull_request:
    branches: [main]

concurrency:
  group: pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  lint-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: '17' }
      - uses: ruby/setup-ruby@v1
        with: { bundler-cache: true }
      - name: Gradle lint + test
        run: ./gradlew :app:lintRelease :app:testReleaseUnitTest
```

### 3. iOS Release Workflow

```yaml
name: Release iOS
on:
  push:
    tags: ['v*.*.*']

jobs:
  ios:
    runs-on: macos-14
    environment: production     # requires manual approval
    timeout-minutes: 60
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with: { bundler-cache: true, working-directory: ios }
      - name: Select Xcode 15.4
        run: sudo xcode-select -s /Applications/Xcode_15.4.app
      - name: Cache CocoaPods
        uses: actions/cache@v4
        with:
          path: ios/Pods
          key: pods-${{ hashFiles('ios/Podfile.lock') }}
      - run: cd ios && pod install --repo-update
      - name: Fastlane release
        working-directory: ios
        env:
          APP_STORE_CONNECT_KEY_ID: ${{ secrets.APP_STORE_CONNECT_KEY_ID }}
          APP_STORE_CONNECT_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_ISSUER_ID }}
          APP_STORE_CONNECT_KEY_CONTENT: ${{ secrets.APP_STORE_CONNECT_KEY_CONTENT }}
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_BASIC_AUTHORIZATION: ${{ secrets.MATCH_GIT_BASIC_AUTHORIZATION }}
          VERSION_NAME: ${{ github.ref_name }}
          VERSION_CODE: ${{ github.run_number }}
        run: |
          bundle exec fastlane beta
          bundle exec fastlane release
      - uses: actions/upload-artifact@v4
        with:
          name: ios-ipa
          path: ios/build/ios/ipa/*.ipa
          retention-days: 30
```

### 4. Android Release Workflow

```yaml
name: Release Android
on:
  push:
    tags: ['v*.*.*']

jobs:
  android:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: '17' }
      - uses: ruby/setup-ruby@v1
        with: { bundler-cache: true, working-directory: android }
      - uses: gradle/actions/setup-gradle@v3
      - name: Decode keystore
        run: |
          echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 -d \
            > android/upload-keystore.jks
          cat > android/key.properties <<EOF
          storeFile=../upload-keystore.jks
          storePassword=${{ secrets.KEYSTORE_PASSWORD }}
          keyAlias=${{ secrets.KEY_ALIAS }}
          keyPassword=${{ secrets.KEY_PASSWORD }}
          EOF
      - name: Fastlane internal
        working-directory: android
        env:
          PLAY_SERVICE_ACCOUNT_JSON: ${{ secrets.PLAY_SERVICE_ACCOUNT_JSON }}
          VERSION_NAME: ${{ github.ref_name }}
        run: bundle exec fastlane internal
      - uses: actions/upload-artifact@v4
        with:
          name: android-aab
          path: android/app/build/outputs/bundle/release/*.aab
```

### 5. Environment Protection

Settings → Environments → `production`:

- Required reviewers: 1 release manager.
- Wait timer: 0 minutes (or 15 for extra caution).
- Deployment branches: only `main` and `release/*` tags.

This gates the release workflow behind a manual click.

### 6. Secret Naming Convention

- Platform prefix: `ANDROID_*`, `IOS_*`, `PLAY_*`, `APP_STORE_CONNECT_*`.
- Encoding suffix: `*_BASE64` when the value is binary-encoded.
- Never: generic names like `API_KEY` or `PASSWORD`.

### 7. Caching Strategy

| Cache | Key |
|-------|-----|
| Ruby gems | `hashFiles('**/Gemfile.lock')` |
| CocoaPods | `hashFiles('**/Podfile.lock')` |
| Gradle | `hashFiles('**/*.gradle*', '**/gradle-wrapper.properties')` via `gradle/actions/setup-gradle` |
| Pub | `hashFiles('**/pubspec.lock')` |
| node_modules | `hashFiles('**/package-lock.json', '**/yarn.lock')` |

Prefer the official setup actions' built-in caching over hand-rolled `actions/cache` where available.

### 8. Artefact Retention

Keep signed AABs and IPAs for 30–90 days. Enough time to re-upload if a release is pulled, short enough to control storage costs. Never store secrets in artefacts.

### 9. Cost Management

- macOS runners are ~10× Linux. Run iOS on macOS only; run Android on Linux.
- Use path filters so docs-only PRs don't spin up mobile CI.
- Cancel in-progress runs on new pushes (`concurrency.cancel-in-progress`).

### 10. Release-Please Integration

```yaml
# .github/workflows/release-please.yml
on: { push: { branches: [main] } }
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        with:
          release-type: simple
```

When the PR merges, a tag is created and the `release.yml` workflow fires. Zero manual steps.

### 11. Anti-Patterns

- Checking signing secrets into `.env` files.
- Triggering release builds on `push` to any branch.
- Running iOS + Android in one job (serialising them wastes macOS time).
- Storing `.p8` or `.jks` files in the repo, even encrypted with git-crypt.

### 12. Checklist
- [ ] Three workflows: PR, main, release — single-purpose each.
- [ ] `production` environment requires manual approval.
- [ ] Secrets named with platform prefix and encoding suffix.
- [ ] macOS runners pinned to a specific Xcode via `xcode-select`.
- [ ] Caches hashed off lockfiles.
- [ ] Fastlane lanes run unchanged; CI YAML only sets env vars.
- [ ] Release-please (or equivalent) closes the loop from merge → tag → release.
