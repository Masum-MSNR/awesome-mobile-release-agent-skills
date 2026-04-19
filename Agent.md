# AGENTS Guidelines for This Repository

This repository contains mobile release engineering conventions for iOS and Android projects. When working on releases interactively with an AI coding agent, please follow the guidelines below to ensure safe, reproducible, policy-compliant shipping.

## 1. Release Specifications
- **iOS Toolchain:** Xcode 15+, Swift 5.9+, minimum deployment iOS 14+ (configurable per app).
- **Android Toolchain:** AGP 8.4+, Kotlin 2.0+, `compileSdk` 34+, `minSdk` 24+ (configurable per app).
- **Automation:** Fastlane 2.220+ is the single source of truth for build/sign/upload; CI is a thin wrapper.
- **Artifact Format:** Android Play uploads AAB (never APK); iOS uploads IPA via App Store Connect API.

## 2. Signing Hygiene
- **Android:** enrol in **Play App Signing**. The repo only holds the upload key; Google holds the app signing key.
- **iOS:** use **`fastlane match`** with a private Git repo (encrypted) or the **App Store Connect API key** (`.p8`). Never commit `.p12`, `.mobileprovision`, or keystores.
- **Secrets storage:** keystores and API keys live in the CI secret store only; local developers use a shared encrypted bucket (1Password, Vault, or match's Git repo).
- **Rotation:** signing passwords and ASC API keys rotate at least yearly; revoked keys are documented in `docs/signing-rotation.md`.

## 3. Reproducible Builds
- **Pinned toolchains:** `.tool-versions` / `mise.toml` / `asdf` pins Xcode, Ruby, Node, Java, and Fastlane.
- **Lockfiles committed:** `Gemfile.lock`, `Podfile.lock`, `yarn.lock` / `package-lock.json`, `gradle/verification-metadata.xml`.
- **Deterministic versions:** `versionCode` / `CFBundleVersion` is derived from CI build number or a monotonic counter; `versionName` / `CFBundleShortVersionString` comes from the Git tag.
- **No network at build time** beyond the initial dependency fetch; offline-capable builds are preferred.

## 4. Fastlane-First Automation
- **Every release action is a lane.** Humans and CI call the same lane; there is no "works on my machine" path.
- **Core lanes:**
  - `fastlane ios beta` — builds, signs, uploads to TestFlight.
  - `fastlane ios release` — submits the latest TestFlight build for review with phased release.
  - `fastlane android internal` — builds AAB, uploads to the Play internal track.
  - `fastlane android promote` — promotes internal → production at 5% staged rollout.
- **`match`, `pilot`/`deliver`, `supply`** handle iOS signing, TestFlight/App Store, and Play uploads respectively.

## 5. Secrets in CI
- **Scoped secrets:** signing secrets are only exposed on the release workflow, not on PRs.
- **Environment protection:** the `production` environment requires manual approval before release lanes run.
- **No secrets in logs:** never `echo $SECRET`; rely on CI secret masking and verify by running the lane with `--verbose` once before shipping.
- **Short-lived credentials:** prefer ASC API keys (`.p8`) over username/password with app-specific passwords.

## 6. Staged Rollouts
- **Android:** every production release starts at **5%** on the Play production track, monitored 24–48h, then 10% → 20% → 50% → 100%.
- **iOS:** enable **phased release for automatic updates** (7-day ramp: 1/2/5/10/20/50/100%).
- **Halt criteria:** crash-free sessions drop below the SLO (see §11), P0 incident, or a policy takedown notice.
- **Never release on Friday afternoons or before holidays** without an on-call owner confirming coverage.

## 7. Feature Flags Separate Release from Exposure
- **Deploy != release.** Code ships behind a flag OFF by default; turning the flag ON is a separate, low-risk action.
- **Kill switches:** every new user-facing feature has a remote flag (LaunchDarkly, Firebase Remote Config, Statsig, or a self-hosted equivalent) evaluated on every session.
- **Flag hygiene:** flags are tagged with an owner and a removal date; stale flags are cleaned up within 30 days of 100% rollout.
- **Cohorting:** risky flags roll out by percentage → country → full; never all-on for 100% of users on day one.

## 8. Rollback Playbooks
- **Android:** halt the staged rollout immediately via Play Console or `fastlane supply --rollout 0`. If the binary is already at 100%, promote the previous AAB from the internal track.
- **iOS:** pause phased release; expedited review request to resubmit the previous build if the regression is severe. Server-side kill switches are always the first line of defense.
- **Communication:** every rollback posts to the `#release` channel with the build number, reason, and ETA for the fix.
- **Post-mortem:** blameless, within 5 business days, with a concrete action item that improves the pipeline.

## 9. Store Policy Compliance
- **iOS:** `PrivacyInfo.xcprivacy` declares Required Reason APIs and tracking domains; App Review metadata (age rating, content descriptions, export compliance) is kept in sync via `deliver`.
- **Android:** the **Data Safety** form in the Play Console matches the SDKs actually shipped; target SDK level is bumped within the grace period Google announces each August.
- **Permissions:** the minimal set; every permission has a rationale string (iOS `NSXxxUsageDescription`, Android runtime permission prompts).
- **Age-gating, payments, and subscriptions** follow the current store rules (StoreKit 2 / Play Billing v7+).

## 10. External Documentation
- When unsure of a store policy or API, consult [App Store Connect API docs](https://developer.apple.com/documentation/appstoreconnectapi), the [Play Developer API](https://developers.google.com/android-publisher), and the [Fastlane docs](https://docs.fastlane.tools). Policy pages (App Store Review Guidelines, Play Developer Program Policies) are authoritative over third-party summaries.

## 11. Release Health SLOs
- **Crash-free sessions:** ≥ 99.5% on both platforms (7-day rolling).
- **Crash-free users:** ≥ 99.0%.
- **ANR rate (Android):** ≤ 0.47% user-perceived (Google's bad-behavior threshold).
- **Breaching any SLO for 30 minutes during a rollout triggers an automatic halt + paging.**

## 12. Useful Agent Skills Recap

| Skill Folder   | Purpose                                                           |
| -------------- | ----------------------------------------------------------------- |
| `signing/`     | Android keystores, iOS certs/profiles, troubleshooting.           |
| `stores/`      | Play Store, App Store Connect, and alternative store publishing.  |
| `rollouts/`    | Staged rollouts, feature flags, iOS phased release.               |
| `versioning/`  | SemVer for mobile, automated release notes.                       |
| `hotfix/`      | Hotfix branches, rollback strategies.                             |
| `ci_cd/`       | Fastlane, Bitrise/Codemagic, GitHub Actions.                      |
| `monitoring/`  | Post-release monitoring, crash SLAs.                              |
| `compliance/`  | iOS privacy manifests, Play Data Safety.                          |

---

Following these practices ensures that mobile releases stay safe, reversible, and policy-compliant. When in doubt, always refer to the specific agent skills provided in `.github/skills/` for deeper task-specific context!

*Note to release engineers: Update this file whenever the team changes signing strategy, rollout policy, or store requirements, so AI agents stay aligned with your conventions.*
