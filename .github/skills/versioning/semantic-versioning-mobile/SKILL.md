---
name: semantic-versioning-mobile
description: Apply SemVer to mobile apps — versionName, versionCode, CFBundleShortVersionString, CFBundleVersion, and release branching. Use this when bumping versions or designing branch strategy.
---

# Semantic Versioning for Mobile

## Instructions

Mobile versioning has **two fields per platform** — a user-visible SemVer string and a monotonic integer. Mix them up and the store rejects your upload. This skill gives you a pattern that maps cleanly to Git tags.

### 1. The Four Fields

| Platform | User-visible | Monotonic integer |
|----------|--------------|-------------------|
| Android | `versionName` (e.g. `1.4.2`) | `versionCode` (e.g. `14020037`) |
| iOS | `CFBundleShortVersionString` (e.g. `1.4.2`) | `CFBundleVersion` (e.g. `14020037`) |

- The user-visible string is SemVer: `MAJOR.MINOR.PATCH`.
- The integer must **strictly increase** for every store upload. Apple allows three components (`14020.37.1`); Play allows a single integer up to 2,100,000,000.

### 2. Computing `versionCode` / `CFBundleVersion`

Option A — tag-derived:

```text
versionCode = MAJOR * 10_000_000 + MINOR * 100_000 + PATCH * 1_000 + buildNumber
# 1.4.2 build 37 → 14020037
```

Option B — CI run number (simpler, fine for most teams):

```text
versionCode = $GITHUB_RUN_NUMBER
```

Pick one and enforce it in CI. Never hand-edit.

### 3. Gradle Wiring

```kotlin
// app/build.gradle.kts
val versionName = findProperty("versionName") as String? ?: "0.0.0"
val versionCode = (findProperty("versionCode") as String?)?.toInt() ?: 1

android {
    defaultConfig {
        this.versionName = versionName
        this.versionCode = versionCode
    }
}
```

Invoke:

```bash
./gradlew :app:bundleRelease \
  -PversionName=$(git describe --tags --abbrev=0 | sed 's/^v//') \
  -PversionCode=$GITHUB_RUN_NUMBER
```

### 4. Xcode Wiring

`Info.plist` uses Xcode variables:

```xml
<key>CFBundleShortVersionString</key><string>$(MARKETING_VERSION)</string>
<key>CFBundleVersion</key><string>$(CURRENT_PROJECT_VERSION)</string>
```

Set at build time:

```bash
xcrun agvtool new-marketing-version 1.4.2
xcrun agvtool new-version -all "$GITHUB_RUN_NUMBER"
```

Or in Fastlane:

```ruby
increment_version_number(version_number: "1.4.2")
increment_build_number(build_number: ENV["GITHUB_RUN_NUMBER"])
```

### 5. SemVer Policy for Mobile

- **MAJOR**: breaking user-visible changes (redesign, new core flow). Rare.
- **MINOR**: new features, backwards-compatible API changes.
- **PATCH**: bug fixes, performance, translation updates.
- **Pre-release suffix**: `1.4.2-beta.1` for TestFlight/internal builds; store prod must not carry a suffix.

### 6. Git Tagging

Tag only production releases:

```bash
git tag -a v1.4.2 -m "Release 1.4.2"
git push --tags
```

CI triggers on `v*.*.*` tags. Pre-production builds are identified by branch + commit SHA only.

### 7. Branching Model

Long-lived branches:

- `main` — always releasable; CI builds internal/TestFlight on every merge.
- `release/1.4.x` — cut from `main` at feature freeze; only bug fixes land here via PR.

Short-lived:

- `feature/<slug>` — branched from `main`, merged via PR.
- `hotfix/<slug>` — branched from the release tag, merged back to both `release/*` and `main`.

Cut a release branch:

```bash
git checkout -b release/1.4.x main
git push -u origin release/1.4.x
# Then bump main to 1.5.0-dev
```

### 8. Handling Rejected Builds

If Apple or Google rejects build N:

1. Bump `CFBundleVersion` / `versionCode` — the previous integer is burned.
2. Keep `versionName` / `CFBundleShortVersionString` the same.
3. Fix and resubmit.

Re-using a rejected build number causes "Redundant Binary" or "Version code already used" errors.

### 9. Version in Code

Expose the version for bug reports and analytics:

```kotlin
val v = "${BuildConfig.VERSION_NAME} (${BuildConfig.VERSION_CODE})"
```

```swift
let v = "\(Bundle.main.infoDictionary?["CFBundleShortVersionString"] ?? "?") "
      + "(\(Bundle.main.infoDictionary?["CFBundleVersion"] ?? "?"))"
```

Include this string in crash reports and support emails.

### 10. Anti-Patterns

- Committing `versionCode` / `CFBundleVersion` to source control and bumping by hand.
- Different SemVer strings on iOS and Android for the same logical release.
- Skipping versions (1.4.0 → 1.6.0) because a build was pulled — confuses users.
- Treating `versionCode` as date-based (`20260419`) without room for same-day hotfixes.

### 11. Checklist
- [ ] Single source of truth for `versionName`: the latest annotated Git tag.
- [ ] `versionCode` / `CFBundleVersion` derived from CI, never hand-edited.
- [ ] iOS and Android ship identical `versionName` for the same commit.
- [ ] Release branching documented; hotfixes merge back to `main`.
- [ ] Version string surfaced in-app on a debug screen and in crash reports.
- [ ] Pre-release suffix (`-beta.1`) used for non-production builds only.
