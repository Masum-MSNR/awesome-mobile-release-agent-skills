---
name: staged-rollouts
description: Use Play percentage rollouts and iOS phased release to limit blast radius. Use this when planning or operating a production release.
---

# Staged Rollouts

## Instructions

A staged rollout is the simplest risk-control tool available to mobile teams. The policy is: **never release to 100% on day one**. Ramp slowly, watch the crash SLO, and halt the moment a regression surfaces.

### 1. Rollout Ladder

Default ladder (adjust per risk level):

| Day | Android (Play) | iOS (phased) |
|-----|----------------|--------------|
| 0 | 5% | 1% (auto) |
| 1 | 10% | 2% (auto) |
| 2 | 20% | 5% (auto) |
| 3 | 50% | 10% (auto) |
| 4–6 | 100% | 20/50/100% (auto) |

Higher-risk releases (new auth flows, IAP changes, framework upgrades) start at 1% and linger 48h per step.

### 2. Play: Start a Staged Rollout

```bash
bundle exec fastlane supply \
  --package_name com.acme.app \
  --aab build/app/outputs/bundle/release/app-release.aab \
  --track production \
  --rollout 0.05 \
  --release_status inProgress \
  --json_key "$PLAY_SERVICE_ACCOUNT_JSON_PATH"
```

### 3. Play: Promote the Percentage

```bash
bundle exec fastlane supply \
  --track production \
  --rollout 0.20 \
  --skip_upload_aab true \
  --skip_upload_metadata true
```

### 4. Play: Halt a Rollout

```bash
# Set rollout to 0 — no new devices receive the new build
bundle exec fastlane supply --track production --rollout 0 --skip_upload_aab true
```

Users already on the new build stay on it; only new installs/updates are paused. For true rollback, re-promote the previous AAB.

### 5. iOS: Enable Phased Release

```ruby
deliver(
  api_key: api_key,
  submit_for_review: true,
  automatic_release: false,
  phased_release: true,
)
```

Apple ramps over 7 days automatically to devices with **auto-updates on**. Users can always manually update to the new version regardless of phase.

### 6. iOS: Pause / Resume / Release-To-All

Via App Store Connect UI or the Connect API:

```bash
# Pause (PATCH to appStoreVersionPhasedReleases)
curl -X PATCH "https://api.appstoreconnect.apple.com/v1/appStoreVersionPhasedReleases/$ID" \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"data":{"type":"appStoreVersionPhasedReleases","id":"'$ID'","attributes":{"phasedReleaseState":"PAUSED"}}}'
```

States: `INACTIVE` → `ACTIVE` → `PAUSED` / `COMPLETE`. You cannot go *back* in phases.

### 7. SLO-Gated Promotion

Don't promote on a schedule; promote on evidence. Automate a gate:

```yaml
# .github/workflows/promote.yml
on:
  schedule: [{ cron: "0 13 * * 1-5" }]   # weekdays 13:00 UTC
jobs:
  check-and-promote:
    runs-on: ubuntu-latest
    steps:
      - name: Fetch crash-free sessions from Firebase
        run: |
          rate=$(curl -s -H "Authorization: Bearer $FIREBASE_TOKEN" \
            "https://crashlytics.googleapis.com/.../metrics/crashFreeSessions?version=1.4.2" \
            | jq -r '.value')
          if (( $(echo "$rate < 99.5" | bc -l) )); then
            echo "Crash-free $rate% below SLO; halting"; exit 1
          fi
      - run: bundle exec fastlane android promote_next
```

### 8. Communication

Every rollout step posts to `#release`:

- Start: version, intended ramp, owner, link to release notes.
- Each step: current %, crash-free rate, notable issues.
- Halt/complete: reason, next action.

### 9. What Not to Ship During a Rollout

- Server-side changes that assume the new client is dominant.
- Schema migrations that break the previous client.
- Feature flags turned on for 100% of users simultaneously with the binary rollout.

Binary rollout and flag flip are always separate; see `feature-flags-release`.

### 10. Rollout Anti-Patterns

- **Friday afternoon releases** without weekend on-call.
- **Skipping steps** because "it looked fine at 10%". Rare bugs surface at scale.
- **Parallel rollouts** of iOS and Android. Stagger by a day so one platform can be a canary for the other.
- **Re-using the same `versionCode`** — Play rejects it. Always bump.

### 11. Checklist
- [ ] Production rollout starts at ≤ 5% (Play) and phased release enabled (iOS).
- [ ] Ladder documented, with a named owner per step.
- [ ] Automated SLO check before each promotion.
- [ ] Halt command rehearsed and tested before the first real use.
- [ ] `#release` channel notified at every transition.
- [ ] Previous binary retained in internal track for quick rollback.
