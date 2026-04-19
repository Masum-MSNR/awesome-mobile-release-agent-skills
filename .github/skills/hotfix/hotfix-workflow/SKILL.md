---
name: hotfix-workflow
description: Run a mobile hotfix — branches, cherry-picks, expedited review, and coordinated Play/App Store submission. Use this when a bug in production needs an urgent patch.
---

# Hotfix Workflow

## Instructions

A hotfix is a minimal, reversible patch applied on top of a released version. It skips the normal feature queue, targets one specific regression, and ships within hours. The priority order is always: **kill switch → hotfix → full release**.

### 1. Decide if It's Actually a Hotfix

Hotfix criteria (all three must hold):

- Impact: widespread (≥ X% of users) or high-severity (data loss, security, payment).
- Server-side mitigation already attempted or impossible.
- Fix is small enough to ship safely within 24h.

Otherwise it's a normal release or a server-side change. Avoid "hotfix hygiene" — using hotfixes to rush features.

### 2. Incident Kickoff Checklist

Within 10 minutes of declaring a hotfix:

- [ ] Incident channel opened, owner assigned.
- [ ] Server-side kill switch flipped if applicable.
- [ ] Play staged rollout halted (`supply --rollout 0`).
- [ ] iOS phased release paused.
- [ ] Customer support informed; macro reply prepared.

### 3. Branch From the Released Tag

```bash
git fetch --tags
git checkout -b hotfix/1.4.3 v1.4.2
```

Never branch a hotfix from `main` — `main` likely contains unreleased work that hasn't been through review.

### 4. Cherry-Pick the Fix

If the fix already merged to `main`:

```bash
git cherry-pick -x <sha>      # -x records the origin SHA in the message
# Resolve conflicts minimally. Do not pull in unrelated changes.
```

If writing a fresh fix, keep it ≤ 30 lines where possible and add a test that fails before the patch.

### 5. Bump Version

```bash
# Patch bump only
./scripts/bump-version.sh 1.4.3   # updates versionName / CFBundleShortVersionString
# versionCode / CFBundleVersion comes from CI run number
```

Tag once CI green:

```bash
git tag -a v1.4.3 -m "Hotfix 1.4.3: <short summary>"
git push origin hotfix/1.4.3 v1.4.3
```

### 6. Ship Android

```bash
bundle exec fastlane android internal    # smoke test
bundle exec fastlane android promote_to_production --rollout 0.10
```

Start higher than the normal 5% because you're already past a bad build — the previous version is worse than the fix. Watch crash SLO for 30–60 min, then ramp to 50% → 100%.

### 7. Ship iOS (Standard Review)

```bash
bundle exec fastlane ios beta            # TestFlight smoke
bundle exec fastlane ios release         # submits with phased release
```

### 8. Ship iOS (Expedited Review)

For P0 issues, request expedited review:

1. Submit via `deliver` as normal.
2. App Store Connect → Resources & Help → Contact Us → App Review → Request Expedited Review.
3. Provide: app name, version, reason (2–3 sentences, factual, include user impact).
4. Apple responds within hours; approval typically within 24h for genuine P0s.

**Do not abuse this** — repeated expedited requests for non-critical issues are tracked and ignored.

### 9. Merge Back to `main`

After the hotfix ships:

```bash
git checkout main
git merge --no-ff hotfix/1.4.3
git push
```

If `main` has diverged significantly, open a PR and resolve conflicts explicitly. The goal is that the hotfix commits are in `main` so the next normal release doesn't regress the fix.

### 10. Parallel Android + iOS Timing

Ideally the fix ships on both platforms within 24h. If only one is broken:

- Ship the fix on the broken platform immediately.
- Include the same fix in the next normal release on the healthy platform — no need to hotfix both.

### 11. Hotfix Template

`HOTFIX_TEMPLATE.md`:

```markdown
## Hotfix 1.4.3
**Summary:**
**Impact:** X% of users, starts version 1.4.2
**Root cause:**
**Fix:** <link to PR>
**Mitigation before fix ships:** <kill switch ref>
**Rollout plan:** Play 10% → monitor 1h → 100%. iOS phased with expedited review.
**Verification:** <test plan>
**Post-mortem owner:**
**Post-mortem due:** <date>
```

### 12. Post-Mortem

Blameless, due within 5 business days. Must include:

- Timeline (detection → mitigation → fix → ship).
- Why the bug escaped CI.
- One concrete process or tooling change to prevent recurrence.
- One action item filed as a ticket with an owner.

### 13. Anti-Patterns

- Cherry-picking a feature flag *and* its code in a hotfix — ship only the minimal patch.
- Skipping TestFlight/internal track for the hotfix build "to save time". Always smoke test.
- Not merging the hotfix back to `main`, causing the bug to reappear in the next release.
- Using expedited review for minor fixes. Save credibility for true emergencies.

### 14. Checklist
- [ ] Hotfix criteria verified; this is not a rushed feature.
- [ ] Kill switch flipped and rollout halted before coding starts.
- [ ] Branch cut from the released tag, not `main`.
- [ ] Minimal diff, with a regression test.
- [ ] Patch `versionName` bump; CI produces fresh integer codes.
- [ ] Smoked on internal/TestFlight before promotion.
- [ ] Merged back to `main` after shipping.
- [ ] Post-mortem scheduled before closing the incident.
