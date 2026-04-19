---
name: crash-rate-sla
description: Define a crash-rate SLO, alert on it, and use breaches as automatic rollback triggers. Use this when formalising release health policy.
---

# Crash Rate SLA

## Instructions

A crash-rate SLA turns "is this release healthy?" from a judgement call into a policy with numeric thresholds. Write it down, alert on it, and let the alert automate rollout decisions.

### 1. Pick the Right Metric

| Metric | Definition | When to use |
|--------|------------|-------------|
| Crash-free sessions | Sessions without a fatal crash / total sessions | Primary SLO — sensitive to short sessions |
| Crash-free users | Users with zero crashes in window / total users | Secondary — reflects user experience |
| ANR rate (Android) | ANRs / sessions | Mandatory on Android |
| App hangs (iOS) | MetricKit hang rate | Complement to crash-free |

Choose **crash-free sessions** as the headline. Crash-free users is more forgiving and lags; sessions catches regressions fastest.

### 2. Target Numbers

Baseline for consumer apps:

- **Crash-free sessions ≥ 99.5%** (7-day rolling, per platform).
- **Crash-free users ≥ 99.0%**.
- **ANR rate ≤ 0.47%** user-perceived (Google's bad behaviour threshold).

Adjust by app risk profile: payments/banking apps should target 99.8%+; casual games can accept 99.0% for novel features behind flags.

### 3. SLO Document

One page, checked into the repo at `docs/release-slo.md`:

```markdown
# Release Health SLO

## Primary
- Crash-free sessions ≥ 99.5%, 7-day rolling, per platform.

## Secondary
- Crash-free users ≥ 99.0%.
- ANR rate ≤ 0.47% (Android).
- Cold start P95 regression ≤ 20% vs previous version.

## Breach Actions
- Any primary SLO < 99.0% for 30 min → PAGE on-call, halt rollout.
- Primary < 99.5% for 2 h → PAGE on-call.
- Secondary breach for 24 h → Slack #release, issue filed.

## Measurement
- Source: Firebase Crashlytics (cross-platform), Play Vitals (Android ANR).
- Scope: current production version only. Beta tracks tracked separately.
```

### 4. Instrument Correctly

- Capture **fatal** exceptions only for the crash-free metric; non-fatals (e.g. Sentry breadcrumbs) are noise.
- Exclude sessions under N seconds that crashed during app init if you separately track launch-crash rate — mixing them obscures trends.
- Tag every event with `app_version`, `build_number`, `device_tier`, `os_version`, `country`.

### 5. Alert Wiring (Sentry)

Sentry Alerts UI or `sentry-cli`:

```bash
sentry-cli alerts create \
  --project mobile-app \
  --name "Crash-Free Sessions Breach" \
  --dataset sessions \
  --query 'release:1.4.2' \
  --aggregate 'crash_free_rate(session)' \
  --comparison '< 0.995' \
  --time-window 30m \
  --action pagerduty:release-oncall
```

### 6. Alert Wiring (Firebase Crashlytics + Cloud Functions)

```javascript
// functions/index.js
exports.onCrashlyticsVelocity = functions.crashlytics.issue()
  .onVelocityAlert(async (issue) => {
    if (issue.appInfo.latestAppVersion !== currentReleaseVersion) return;
    await pagerduty.trigger({
      summary: `Velocity alert on ${issue.issueTitle}`,
      severity: 'critical',
      component: 'mobile-release',
    });
  });
```

Firebase's "velocity alert" fires when a single issue rises above a threshold share of sessions in a short window — often earlier than the aggregate SLO.

### 7. Automated Rollout Halt

Wire the alert action to a webhook that halts the Play rollout:

```bash
#!/usr/bin/env bash
# halt-rollout.sh — invoked by PagerDuty webhook on SLO breach
set -euo pipefail

cd "$(dirname "$0")/.."
bundle exec fastlane supply \
  --track production --rollout 0 \
  --skip_upload_aab true --skip_upload_metadata true \
  --json_key_data "$PLAY_SERVICE_ACCOUNT_JSON"

curl -X POST "$SLACK_WEBHOOK" \
  -d "{\"text\":\":rotating_light: Auto-halted Play rollout due to SLO breach\"}"
```

### 8. iOS Phased Release Halt

```bash
#!/usr/bin/env bash
JWT=$(./scripts/asc-jwt.sh)
ID=$(curl -s "https://api.appstoreconnect.apple.com/v1/appStoreVersions/$VERSION_ID/appStoreVersionPhasedRelease" \
  -H "Authorization: Bearer $JWT" | jq -r .data.id)
curl -X PATCH "https://api.appstoreconnect.apple.com/v1/appStoreVersionPhasedReleases/$ID" \
  -H "Authorization: Bearer $JWT" -H "Content-Type: application/json" \
  -d '{"data":{"type":"appStoreVersionPhasedReleases","id":"'"$ID"'",
       "attributes":{"phasedReleaseState":"PAUSED"}}}'
```

### 9. Error Budgets

Compute monthly error budget: `budget_minutes = (1 - 0.995) × 30 × 24 × 60 ≈ 216 minutes` of sub-SLO time per month. Track burn rate; sustained high burn rate justifies slowing feature velocity to invest in stability.

### 10. False Positives

Guard against:

- **SDK-side reporting outages** (Crashlytics dropped events, Sentry throttling). Cross-verify with a second source before paging.
- **Low-traffic cohorts** (new country rollout with 200 sessions a day). A 99.5% threshold on 200 sessions is statistical noise. Alert only when `sessions > 10_000` in the window.
- **Known device bugs** (a specific OEM-OS combo) that dominate new crashes but are not your code. Filter before alerting.

### 11. Anti-Patterns

- Setting SLO based on peer app rumour rather than historical data.
- Measuring crash-free across all versions — new release regressions get hidden by stable old versions.
- No automated halt — relying on a human to see the page and act within minutes.
- Tuning SLO down because alerts are noisy; fix the noise, keep the target.

### 12. Checklist
- [ ] `docs/release-slo.md` written, reviewed, and agreed with product.
- [ ] Primary and secondary metrics instrumented on both platforms.
- [ ] Alerts configured with per-version filter and minimum sample size.
- [ ] Automated halt script tested in staging.
- [ ] Error budget tracked monthly; burn rate visible in dashboards.
- [ ] On-call runbook includes the page-to-halt path step-by-step.
- [ ] SLO reviewed quarterly as historical data matures.
