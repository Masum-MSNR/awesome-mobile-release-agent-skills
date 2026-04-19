---
name: feature-flags-release
description: Decouple deploy from release using feature flags, kill switches, and remote config. Use this when shipping risky or incremental features.
---

# Feature Flags for Release

## Instructions

The core principle: **deploy ≠ release**. Ship code behind flags turned OFF; turn them ON as a separate, small, reversible action. Every new user-facing capability needs a flag *and* a kill switch from day one.

### 1. Flag Taxonomy

| Kind | Purpose | Lifetime |
|------|---------|----------|
| Release flag | Gate unfinished work in main | Days–weeks, then removed |
| Experiment flag | A/B test a variant | Weeks, then winner picked |
| Ops / kill switch | Emergency disable of a feature or SDK | Permanent |
| Permission flag | Paid/entitlement gating | Permanent |

Mixing kinds in one flag leads to dead code paths. Keep them separate.

### 2. Choosing a Flag System

- **LaunchDarkly / Statsig / Unleash / ConfigCat** — rich targeting, audit logs, SDKs for iOS/Android.
- **Firebase Remote Config** — free, simpler model; default-value fallback on every get.
- **Self-hosted** — fine for kill switches only; don't build targeting yourself.

All decent systems evaluate on-device with a cached ruleset and refresh in the background.

### 3. Wiring a Flag (Kotlin)

```kotlin
interface FeatureFlags {
    fun isEnabled(key: String, default: Boolean = false): Boolean
    fun variant(key: String, default: String = "control"): String
}

class RemoteConfigFlags(private val rc: FirebaseRemoteConfig) : FeatureFlags {
    override fun isEnabled(key: String, default: Boolean) =
        if (rc.getKeyExists(key)) rc.getBoolean(key) else default
    override fun variant(key: String, default: String) =
        rc.getString(key).ifEmpty { default }
}
```

### 4. Wiring a Flag (Swift)

```swift
protocol FeatureFlags {
    func isEnabled(_ key: String, default: Bool) -> Bool
    func variant(_ key: String, default: String) -> String
}

final class StatsigFlags: FeatureFlags {
    func isEnabled(_ key: String, default d: Bool) -> Bool {
        Statsig.checkGate(key) // defaults to false if not loaded
    }
    func variant(_ key: String, default d: String) -> String {
        Statsig.getExperiment(key).getValue(forKey: "variant", defaultValue: d) as? String ?? d
    }
}
```

### 5. Kill Switch Pattern

Every network-dependent feature checks a kill switch before its first call:

```kotlin
if (flags.isEnabled("checkout_v2_enabled", default = false)) {
    startCheckoutV2()
} else {
    startCheckoutLegacy()
}
```

The default is the *safe* state — usually "feature OFF". If remote config fails to load, users get the old, proven path.

### 6. Rollout Targeting

Roll out by:

1. Internal team (email domain) — always first.
2. Percentage of users (1% → 5% → 25% → 100%).
3. App version (`>= 1.4.2` to avoid enabling on versions that don't support it).
4. Geography (launch in low-traffic markets first).

Never target "new installs only" for a kill switch — you need to cover the whole base.

### 7. Flag Hygiene

- Tag every flag with **owner** and **removal date**.
- CI lints for flags older than 60 days past their removal date.
- When a flag reaches 100% and is stable for 30 days, delete both the flag and the dead code path in the same PR.

Example lint:

```bash
yq eval '.flags[] | select(.removal_date < now) | .key' flags.yaml
```

### 8. Relationship to Binary Rollout

Binary rollout and flag flip are **independent axes**:

| Binary | Flag | Meaning |
|--------|------|---------|
| Old (1.4.1) at 95% | OFF | No users see the feature. |
| New (1.4.2) at 5% | OFF | 5% have the *code*; none see it. |
| New (1.4.2) at 100% | OFF | Entire base has the code; still dark. |
| New (1.4.2) at 100% | 10% | Real exposure begins. |

This separation is what makes rollbacks fast: flip the flag, don't touch the store.

### 9. Testing With Flags

- **Unit:** inject a fake `FeatureFlags` that returns the variant under test.
- **UI:** override flag values via a debug menu (`DebugFlagsScreen`) gated by a build flavor.
- **CI:** snapshot both ON and OFF variants for every flagged flow.

### 10. Observability

Log `flag.key`, `flag.value`, `flag.source` (`remote`/`default`/`override`) on every evaluation that gates a user-visible surface. This lets you reconstruct what a user saw during a regression.

```kotlin
analytics.logEvent("flag_eval",
    "key" to key, "value" to value, "source" to source, "user_id" to userId)
```

### 11. Anti-Patterns

- **Forever-flags**: no owner, no removal date. They become the config system by accident.
- **Flags controlled by App Review state** (turning features on only after review passes). Use TestFlight/internal track instead.
- **Branching on flags deep in business logic**. Prefer evaluating once at an entry point and passing the result as state.

### 12. Checklist
- [ ] Every new feature has a flag, OFF by default, with a safe fallback.
- [ ] Every remote-dependent feature has a kill switch evaluated on launch.
- [ ] Flag system picked and SDK integrated; cached for offline use.
- [ ] Flag registry (`flags.yaml`) with owner + removal date per flag.
- [ ] CI lint fails on expired flags.
- [ ] Flag evaluations logged to analytics with source.
- [ ] Binary rollout and flag flip treated as independent release steps.
