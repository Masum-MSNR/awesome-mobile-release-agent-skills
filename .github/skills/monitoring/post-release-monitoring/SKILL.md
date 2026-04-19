---
name: post-release-monitoring
description: Monitor a mobile release — crash-free sessions SLO, adoption curve, ratings, and performance regressions. Use this during the first week after every release.
---

# Post-Release Monitoring

## Instructions

A release isn't "done" when the build hits the store. The first 72 hours are when regressions surface and 90% of rollback decisions are made. This skill defines what to watch, where to watch it, and when to halt.

### 1. The Four Signals

Watch these in a single dashboard, filtered to the current version:

1. **Crash-free sessions** (primary SLO).
2. **Adoption curve** — are users actually installing the new build?
3. **Store ratings** — a 1-star spike often leads SLO breaches by hours.
4. **Key business metrics** — DAU, session length, conversion, revenue.

### 2. Data Sources

| Signal | iOS | Android |
|--------|-----|---------|
| Crash-free | Firebase Crashlytics, Sentry, App Store Connect Metrics | Same + Play Vitals |
| ANR | n/a | Play Vitals + Firebase |
| Adoption | App Store Connect Analytics | Play Console Statistics |
| Ratings | App Store Connect | Play Console Ratings |
| Perf | Firebase Performance, MetricKit | Firebase, Play Vitals |

### 3. Crash-Free Sessions SLO

Target: **≥ 99.5% 7-day rolling**. Breach criteria:

- < 99.0% at any single 30-minute window during rollout → halt.
- < 99.5% sustained over 2 hours → halt.
- New crash type with > 1000 events in 24h → halt regardless of rate.

### 4. Adoption Curve

Expected adoption per day of phased release (iOS):

| Day | Cumulative users on new version |
|-----|-------------------------------|
| 1 | ~1% |
| 3 | ~5% |
| 5 | ~20% |
| 7 | ~70% |
| 14 | ~90% |

Play: 50–70% adoption by day 7 is typical. Deviations below signal either a distribution bug (build marked ineligible by the store) or a flag/permission that blocks auto-update.

### 5. Ratings Surveillance

Rating drop > 0.1 within 48h often precedes an SLO breach. Categorise reviews by topic:

```python
# Simple keyword categorisation
categories = {
    "crash":   ["crash", "freeze", "won't open", "force close"],
    "login":   ["can't log in", "login", "sign in", "authentication"],
    "payment": ["payment", "charge", "subscription", "refund"],
    "perf":    ["slow", "laggy", "battery", "drain"],
}
```

Surface the daily top-3 complaint topics to `#release`.

### 6. ANR (Android-only)

Google's **bad behaviour threshold** is 0.47% user-perceived ANR. Above this your app can be featured-demoted or warned in Play Console. Watch per-version in Play Vitals.

### 7. Performance Regressions

Track P50 / P95 cold-start time and key interactive transactions (e.g. add-to-cart). Regressions > 20% vs the previous version are release-blocking even if crashes are flat.

```kotlin
// Firebase Performance custom trace
val trace = Firebase.performance.newTrace("checkout_submit")
trace.start()
// ... work ...
trace.putMetric("item_count", cart.items.size.toLong())
trace.stop()
```

### 8. Alerting

Alert on the signal, not the data source. Consolidate to PagerDuty / Opsgenie:

```yaml
alerts:
  - name: crash_free_sessions_p0
    condition: crash_free_sessions < 99.0 for 30m on current_version
    severity: P0
    page: release-oncall
  - name: anr_rate_warning
    condition: anr_rate > 0.40 for 6h on current_version
    severity: P1
    notify: "#release"
  - name: rating_drop
    condition: avg_rating - avg_rating_lag_7d < -0.15
    severity: P1
```

### 9. MetricKit (iOS)

Opt in to `MetricKit` payloads for real-user perf and crash diagnostics:

```swift
final class MetricsObserver: NSObject, MXMetricManagerSubscriber {
    override init() {
        super.init()
        MXMetricManager.shared.add(self)
    }
    func didReceive(_ payloads: [MXMetricPayload]) {
        payloads.forEach { upload($0.jsonRepresentation()) }
    }
    func didReceive(_ payloads: [MXDiagnosticPayload]) {
        payloads.forEach { upload($0.jsonRepresentation()) }
    }
}
```

Forward to your backend for aggregation across users.

### 10. Dashboard Layout

One screen, five widgets:

1. Crash-free sessions line (current vs previous 2 versions).
2. Rollout % vs adoption %.
3. Top 5 crashes (with `first_seen = current_version`).
4. Ratings timeline + top-3 complaint topics.
5. Key business metric vs 28-day baseline.

Share the dashboard URL in the release announcement.

### 11. Monitoring Window

- **T+0 to T+2h:** engineer watches live.
- **T+2h to T+24h:** on-call rotation checks every 2h.
- **T+24h to T+7d:** daily standup review.
- **T+7d:** release retro; incorporate findings into CI checks.

### 12. Checklist
- [ ] Dashboard exists and is bookmarked; URL in the release announcement.
- [ ] SLO (99.5% crash-free 7d) codified as an alert.
- [ ] ANR alert configured for Android.
- [ ] Adoption curve compared against expected ramp; anomalies investigated.
- [ ] Ratings + review sentiment checked daily for 7 days.
- [ ] Perf P95 tracked for critical flows; regression alerts configured.
- [ ] On-call rotation documented; handoffs logged.
