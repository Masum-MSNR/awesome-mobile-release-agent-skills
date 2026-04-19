---
name: rollback-strategy
description: Halt rollouts, restore previous binaries, and use kill switches for server-side rollback. Use this when a production release is causing harm.
---

# Rollback Strategy

## Instructions

Mobile rollbacks are fundamentally harder than web rollbacks: you cannot remove a binary that users have already installed. The order of preference is always **(1) server-side kill switch, (2) halt rollout, (3) ship a new binary with the fix, (4) re-promote the previous binary**.

### 1. Rollback Decision Tree

```text
Is there a feature flag covering the regression?
 ├─ YES → Flip the flag OFF. Done in seconds. Fix forward later.
 └─ NO → Is the server API the root cause?
          ├─ YES → Roll back the server change. Done in minutes.
          └─ NO → Is the rollout still < 100%?
                   ├─ YES → Halt the rollout and hotfix (see `hotfix-workflow`).
                   └─ NO → Hotfix is the only path. Users on the bad version stay there
                           until they manually update or auto-update triggers.
```

### 2. Kill Switches First

Every new feature should ship with a kill switch (see `feature-flags-release`). A kill switch is:

- Evaluated on every session start.
- Has a safe default (usually OFF).
- Documented in the PR that introduces the feature.

Flip via LaunchDarkly / Statsig / Remote Config console. Log the change with who, when, and why.

### 3. Halt a Play Rollout

```bash
bundle exec fastlane supply \
  --track production \
  --rollout 0 \
  --skip_upload_aab true \
  --skip_upload_metadata true
```

Effect: no *new* installs or updates receive the new build. Users already on it stay on it.

### 4. Pause an iOS Phased Release

```bash
curl -X PATCH \
  "https://api.appstoreconnect.apple.com/v1/appStoreVersionPhasedReleases/$ID" \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"data":{"type":"appStoreVersionPhasedReleases","id":"'"$ID"'",
       "attributes":{"phasedReleaseState":"PAUSED"}}}'
```

Pause pauses the auto-update ramp. Manual updaters still get the paused build.

### 5. "Re-Promote" the Previous Binary (Android)

Play does not support rollback to a lower `versionCode`. Instead, you re-ship the previous source as a new build:

```bash
git checkout v1.4.1
./scripts/bump-version.sh 1.4.1.1   # bump patch to avoid the burned code
bundle exec fastlane android internal
bundle exec fastlane android promote_to_production --rollout 0.20
```

Keep the previous AAB in the internal track so you can re-promote quickly without rebuilding.

### 6. "Rollback" on iOS

Same principle: resubmit the previous source as a new higher `CFBundleVersion`. Request expedited review because the existing version is actively harming users.

### 7. Partial Rollback with Server Steering

If only a subset of features is broken, steer clients server-side:

```jsonc
// /config endpoint
{
  "minimum_supported_version": "1.4.3",
  "checkout_v2_enabled": false,
  "force_update_below_version": "1.4.0"
}
```

The client honours `minimum_supported_version` by hiding broken flows or prompting the user to update.

### 8. Force-Update Prompts (Use Sparingly)

Android:

```kotlin
val updateInfo = appUpdateManager.appUpdateInfo.await()
if (updateInfo.updateAvailability() == UpdateAvailability.UPDATE_AVAILABLE &&
    updateInfo.isUpdateTypeAllowed(AppUpdateType.IMMEDIATE) &&
    serverConfig.forceUpdateBelow > BuildConfig.VERSION_CODE) {
    appUpdateManager.startUpdateFlowForResult(updateInfo, AppUpdateType.IMMEDIATE, activity, REQ)
}
```

iOS: there is no native force-update; implement it as a full-screen gate that only dismisses after the user taps "Update" and returns from the App Store.

### 9. Data Migrations on Rollback

If the bad release shipped a migration that rewrites local data, rolling back the binary does **not** roll back the data. Design migrations to be:

- **Additive** (new columns, not dropping old ones).
- **Versioned** (`schema_version` tracked).
- **Tolerant of being seen by both N and N-1 clients** for a transition period.

If a migration is destructive, ship it behind a flag and enable only after the binary is at 100% stable.

### 10. Communication Template

```text
[ROLLBACK] app-release 1.4.2
Trigger: crash-free sessions dropped to 98.7% (SLO 99.5%)
Action: Play rollout halted at 10%; iOS phased paused.
Users affected: ~60k on 1.4.2.
Mitigation: server-side flag `checkout_v2_enabled` = false.
Next step: hotfix 1.4.3 ETA 6h; owner @alice.
```

Post to `#release`, `#support`, and `#engineering`.

### 11. What Not to Do

- **Do not publish a lower `versionCode`** — it will be rejected and burn a number.
- **Do not pull the app from the store** as a rollback tactic. It confuses users and loses reviews.
- **Do not leave a halted rollout in place for days.** Either resume or ship a hotfix; users on the bad version need the fix.
- **Do not pretend a rollback didn't happen** — be transparent in release notes.

### 12. Rehearsals

Rollback commands must be muscle memory. Run a quarterly drill in a staging project:

1. Submit a known-bad build to internal track.
2. Practice halting, pausing, flipping flags, promoting the previous binary.
3. Time each step. Target: kill switch < 2 min, halt rollout < 5 min.

### 13. Checklist
- [ ] Every feature has a kill switch or remote config gate.
- [ ] Server-side steering endpoint (`/config`) in place and cached client-side.
- [ ] Previous binary retained in Play internal and TestFlight for quick re-promotion.
- [ ] Halt + pause commands scripted and runnable with one command.
- [ ] Data migrations designed to tolerate N/N-1 coexistence.
- [ ] Rollback rehearsed within the last 90 days.
- [ ] Communication template adapted and saved in `docs/incident-templates/`.
