# Awesome Mobile Release Agent Skills

[![Agent Skills](https://img.shields.io/badge/Agent-Skills-blue?style=flat&logo=github)](https://agentskills.io)
[![Release Engineering](https://img.shields.io/badge/Release-Engineering-orange?style=flat)](#)
[![Fastlane](https://img.shields.io/badge/Fastlane-2.220+-green?style=flat&logo=fastlane)](https://fastlane.tools)
[![App Store](https://img.shields.io/badge/App%20Store-Connect-black?style=flat&logo=apple)](https://appstoreconnect.apple.com)
[![Play Store](https://img.shields.io/badge/Play%20Store-Console-brightgreen?style=flat&logo=googleplay)](https://play.google.com/console)
[![License: MIT](https://img.shields.io/badge/Code-MIT-green?style=flat)](LICENSE-CODE)
[![License: CC BY 4.0](https://img.shields.io/badge/Docs-CC%20BY%204.0-lightgrey?style=flat)](LICENSE-DOCS)

A collection of **20 agent skills** that give AI coding assistants (GitHub Copilot, Claude, Gemini, Cursor, etc.) expert-level knowledge of modern mobile release engineering across iOS and Android.

Drop the `.github/skills/` folder into your project and your agent automatically follows release best practices, uses current store policies, and ships builds that pass review.

> Learn more about the Agent Skills standard at [agentskills.io](https://agentskills.io).

---

## Why use this

- **Store-aware** -- skills cover Play Console, App Store Connect, TestFlight, and alternative stores (Amazon, Huawei, Samsung) as first-class targets.
- **Fastlane-first** -- every automation lane is expressed as a `Fastfile` first, with GitHub Actions / Bitrise / Codemagic as thin wrappers.
- **Safe by default** -- staged rollouts, feature flags, and kill switches decouple deploy from exposure; rollback is always pre-planned.
- **Actionable checklists** -- every skill ends with a verification checklist the agent can self-audit against before a release ships.

---

## How it works

The `Agent.md` file is the foundation. Every AI agent reads it first to understand your release conventions. Individual skills layer on top for specific tasks.

```text
       ┌───────────────────────────────────────────┐
       │                  Agent.md                 │
       │     (read by all AI agents implicitly)    │
       └─────────────────────┬─────────────────────┘
                             │
     ┌───────────────┬───────┴───────┬───────────────┐
     ▼               ▼               ▼               ▼
┌──────────┐   ┌───────────┐   ┌────────────┐  ┌────────────┐
│ Signing  │   │  Stores   │   │ Rollouts   │  │   CI/CD    │
├──────────┤   ├───────────┤   ├────────────┤  ├────────────┤
│android   │   │play       │   │staged      │  │fastlane    │
│ios       │   │app-store  │   │flags       │  │bitrise     │
│trouble   │   │alternative│   │phased-ios  │  │gh-actions  │
└──────────┘   └───────────┘   └────────────┘  └────────────┘

┌──────────┐   ┌───────────┐   ┌────────────┐  ┌────────────┐
│Versioning│   │  Hotfix   │   │ Monitoring │  │ Compliance │
├──────────┤   ├───────────┤   ├────────────┤  ├────────────┤
│semver    │   │workflow   │   │post-release│  │privacy-ios │
│notes     │   │rollback   │   │crash-sla   │  │data-safety │
└──────────┘   └───────────┘   └────────────┘  └────────────┘
```

---

## Available Skills

All skills live in `.github/skills/<category>/<skill-name>/SKILL.md`.

### Signing

| Skill | What it covers |
|-------|----------------|
| [Android Signing](.github/skills/signing/android-signing/SKILL.md) | Keystore generation, Play App Signing, v1/v2/v3/v4 signature schemes |
| [iOS Signing & Provisioning](.github/skills/signing/ios-signing-provisioning/SKILL.md) | Certificates, provisioning profiles, automatic vs manual, `match` |
| [Code Signing Troubleshooting](.github/skills/signing/code-signing-troubleshooting/SKILL.md) | Common Apple and Google signing errors and their fixes |

### Stores

| Skill | What it covers |
|-------|----------------|
| [Play Store Release](.github/skills/stores/play-store-release/SKILL.md) | Internal/closed/open tracks, AAB upload, Data Safety form |
| [App Store Release](.github/skills/stores/app-store-release/SKILL.md) | App Store Connect, TestFlight, metadata, screenshots |
| [Alternative Stores](.github/skills/stores/alternative-stores/SKILL.md) | Amazon, Huawei AppGallery, Samsung Galaxy Store, HMS |

### Rollouts

| Skill | What it covers |
|-------|----------------|
| [Staged Rollouts](.github/skills/rollouts/staged-rollouts/SKILL.md) | Play percentage rollouts, phased release on iOS |
| [Feature Flags for Release](.github/skills/rollouts/feature-flags-release/SKILL.md) | Decoupling deploy from exposure, kill switches |
| [Phased Release (iOS)](.github/skills/rollouts/phased-release-ios/SKILL.md) | Apple phased release behavior, pausing, resuming |

### Versioning

| Skill | What it covers |
|-------|----------------|
| [Semantic Versioning for Mobile](.github/skills/versioning/semantic-versioning-mobile/SKILL.md) | `versionName` / `versionCode` / `CFBundleVersion`, branching |
| [Release Notes Automation](.github/skills/versioning/release-notes-automation/SKILL.md) | `git-cliff`, conventional-changelog, release-please |

### Hotfix

| Skill | What it covers |
|-------|----------------|
| [Hotfix Workflow](.github/skills/hotfix/hotfix-workflow/SKILL.md) | Hotfix branches, cherry-picks, expedited review requests |
| [Rollback Strategy](.github/skills/hotfix/rollback-strategy/SKILL.md) | Halting rollouts, previous binaries, kill switches |

### CI / CD

| Skill | What it covers |
|-------|----------------|
| [Fastlane Setup](.github/skills/ci_cd/fastlane-setup/SKILL.md) | `Fastfile` structure, lanes, `match`, `pilot`, `supply` |
| [Bitrise & Codemagic](.github/skills/ci_cd/bitrise-codemagic/SKILL.md) | Stack choice, workflows, caches |
| [GitHub Actions for Mobile Release](.github/skills/ci_cd/github-actions-mobile-release/SKILL.md) | macOS runners, AAB/IPA artifacts, signing secrets |

### Monitoring

| Skill | What it covers |
|-------|----------------|
| [Post-Release Monitoring](.github/skills/monitoring/post-release-monitoring/SKILL.md) | Crash-free sessions SLO, adoption, ratings |
| [Crash Rate SLA](.github/skills/monitoring/crash-rate-sla/SKILL.md) | SLO definition, alerting, rollback triggers |

### Compliance

| Skill | What it covers |
|-------|----------------|
| [Privacy Manifests (iOS)](.github/skills/compliance/privacy-manifests-ios/SKILL.md) | Required Reason APIs, `PrivacyInfo.xcprivacy` |
| [Play Data Safety](.github/skills/compliance/play-data-safety/SKILL.md) | Data Safety form, declarations, audits |

---

## Quick Start

1. **Copy** the `.github/skills/` folder and `Agent.md` to your project root.
2. **Verify** the structure:
   ```text
   my-mobile-project/
   ├── Agent.md
   ├── .github/
   │   └── skills/
   │       ├── signing/
   │       │   ├── android-signing/
   │       │   │   └── SKILL.md
   │       │   └── ...
   │       └── ...
   ├── ios/
   ├── android/
   └── ...
   ```
3. **Reload** your editor (Copilot and similar extensions may need a window reload to index new skills).

Then just ask your agent naturally:

> "Set up a TestFlight lane with App Store Connect API keys."
> "Start a 5% staged rollout on the Play internal track."

The agent picks the right skill automatically.

---

## Other Environments

- **Claude**: copy or symlink `.github/skills` to `.claude/skills/`.
- **OpenCode**: supports `.opencode/skill/` and `.claude/skills/`. Global skills go in `~/.config/opencode/skill/`.

---

## Adding Custom Skills

Create a folder in `.github/skills/` with a `SKILL.md` containing the required frontmatter:

```markdown
---
name: my-custom-skill
description: What this skill does
---
# Instructions
...
```

---

## License

Dual-licensed:

- **Code** (snippets, configs, scripts) -- [MIT](LICENSE-CODE)
- **Documentation** (SKILL.md, README, Agent.md) -- [CC BY 4.0](LICENSE-DOCS)

See [LICENSE](LICENSE) for details.
