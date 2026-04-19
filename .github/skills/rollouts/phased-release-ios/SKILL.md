---
name: phased-release-ios
description: Use Apple's phased release for automatic updates on iOS — behavior, pausing, resuming, and interaction with manual updates. Use this when shipping production iOS builds.
---

# iOS Phased Release

## Instructions

Apple's **Phased Release for Automatic Updates** throttles how many auto-updating devices receive a new version over a 7-day window. It's mandatory for safe iOS rollouts and complements (does not replace) server-side feature flags.

### 1. How the 7-Day Ramp Works

| Day | Auto-update devices that can receive | Cumulative |
|-----|---------------------------------------|------------|
| 1 | 1% | 1% |
| 2 | 2% | 2% |
| 3 | 5% | 5% |
| 4 | 10% | 10% |
| 5 | 20% | 20% |
| 6 | 50% | 50% |
| 7 | 100% | 100% |

**Important:** any user who manually updates (App Store → search → Update) gets the new version immediately, regardless of phase. Phased release only affects auto-updates.

### 2. Enable with `deliver`

```ruby
lane :release do
  api_key = app_store_connect_api_key(...)
  deliver(
    api_key: api_key,
    submit_for_review: true,
    automatic_release: false,
    phased_release: true,
    force: true,
  )
end
```

Alternatively enable in App Store Connect → My App → Version → Phased Release for Automatic Updates → **Release update over 7-day period**.

### 3. States

- **INACTIVE** — not started (version in review or pending developer release).
- **ACTIVE** — ramp in progress.
- **PAUSED** — ramp halted at the current percentage; developer action required.
- **COMPLETE** — 100% of auto-updaters covered.

State transitions are one-way forward (`ACTIVE → PAUSED → ACTIVE → COMPLETE`); you cannot rewind.

### 4. Pause the Ramp

Via UI (Connect → version → Pause Phased Release) or via the App Store Connect API:

```bash
curl -X PATCH \
  "https://api.appstoreconnect.apple.com/v1/appStoreVersionPhasedReleases/$ID" \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "type": "appStoreVersionPhasedReleases",
      "id": "'"$ID"'",
      "attributes": { "phasedReleaseState": "PAUSED" }
    }
  }'
```

Fetch the `$ID`:

```bash
curl -s "https://api.appstoreconnect.apple.com/v1/appStoreVersions/$VERSION_ID/appStoreVersionPhasedRelease" \
  -H "Authorization: Bearer $JWT" | jq -r .data.id
```

### 5. Resume the Ramp

Set `phasedReleaseState` back to `ACTIVE`. The schedule resumes from the current step, not the start.

### 6. Release to All Users

If a build is healthy and you want to skip the remaining days:

```bash
curl -X PATCH \
  "https://api.appstoreconnect.apple.com/v1/appStoreVersionPhasedReleases/$ID" \
  -d '{"data":{"type":"appStoreVersionPhasedReleases","id":"'"$ID"'",
       "attributes":{"phasedReleaseState":"COMPLETE"}}}'
```

### 7. Interaction with Rollback

Pausing the phase does **not** pull the binary from users who already received it. For a true rollback you must:

1. Pause the ramp.
2. Submit the previous source as a new build (you cannot resubmit the same binary).
3. Request expedited review.
4. Users on the regressed build update to the new (formerly old) version.

Server-side kill switches via feature flags are always faster than any store-level rollback.

### 8. When to Disable Phased Release

Rare but legitimate cases:

- **Security hotfix** where every hour of exposure is expensive — disable phased, release to all at once.
- **Compliance deadline** (GDPR, tax, age-gating) that requires 100% rollout on a specific date.
- **App uses server-side feature flags** with per-user rollout already, so the binary can be broadcast.

Disable by unchecking the option in Connect or passing `phased_release: false`.

### 9. Monitoring During a Phased Release

Track by day:

- Crash-free sessions at the current phase vs the previous version.
- New crashes not present on the prior version (`first_seen = this_version`).
- Adoption curve matches Apple's expected ramp; deviations suggest auto-update disabled on a cohort.
- App Store ratings on this version.

Integrate with Firebase Crashlytics, Sentry, or App Store Connect's Metrics API.

### 10. Metrics API Sample

```bash
curl -s "https://api.appstoreconnect.apple.com/v1/apps/$APP_ID/perfPowerMetrics" \
  -H "Authorization: Bearer $JWT" -H "Accept: application/vnd.apple.xcode-metrics+json"
```

### 11. Checklist
- [ ] `phased_release: true` set on every production `deliver` lane.
- [ ] Pause/resume commands rehearsed; JWT generation scripted.
- [ ] Server-side kill switches in place for features where phased release is too slow.
- [ ] Monitoring dashboard filtered to the current version for the full 7 days.
- [ ] Manual-update bypass understood and communicated to stakeholders.
- [ ] Decision documented whenever phased release is skipped, with justification.
