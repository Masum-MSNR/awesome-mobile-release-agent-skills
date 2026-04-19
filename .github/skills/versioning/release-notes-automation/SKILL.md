---
name: release-notes-automation
description: Generate release notes from Conventional Commits with git-cliff, conventional-changelog, or release-please. Use this when automating the changelog and store release notes.
---

# Release Notes Automation

## Instructions

Release notes should be generated, not written from memory. Conventional Commits + a generator produces three artefacts automatically: a project `CHANGELOG.md`, store release notes, and an in-app "What's New" screen.

### 1. Adopt Conventional Commits

Commit format:

```text
<type>(<scope>)!: <description>

[body]

[footer, e.g. BREAKING CHANGE: ..., Refs: JIRA-123]
```

Common types: `feat`, `fix`, `perf`, `refactor`, `docs`, `chore`, `test`, `build`, `ci`.

Enforce with `commitlint`:

```json
// commitlint.config.js
{
  "extends": ["@commitlint/config-conventional"],
  "rules": { "subject-case": [2, "always", "sentence-case"] }
}
```

Add a `commit-msg` Git hook with Husky or Lefthook.

### 2. Pick a Generator

| Tool | Strengths | Use when |
|------|-----------|----------|
| `git-cliff` | Single Rust binary, Keep-a-Changelog format, templated | You want zero JS deps |
| `conventional-changelog-cli` | Node ecosystem, many presets | You already live in npm |
| `release-please` | PR-based, bumps versions, files tags | You want "release on merge" automation |

### 3. `git-cliff` Setup

`cliff.toml`:

```toml
[changelog]
header = "# Changelog\n\nAll notable changes to this project will be documented in this file.\n"
body = """
{% if version %}## [{{ version }}] - {{ timestamp | date(format="%Y-%m-%d") }}{% endif %}
{% for group, commits in commits | group_by(attribute="group") %}
### {{ group | upper_first }}
{% for commit in commits %}
- {{ commit.message | upper_first }}{% if commit.breaking %} (**BREAKING**){% endif %}
{% endfor %}
{% endfor %}
"""

[git]
conventional_commits = true
filter_unconventional = true
commit_parsers = [
    { message = "^feat",       group = "Features" },
    { message = "^fix",        group = "Bug Fixes" },
    { message = "^perf",       group = "Performance" },
    { message = "^refactor",   group = "Refactor" },
    { body    = ".*BREAKING",  group = "Breaking" },
    { message = "^(chore|docs|test|ci|build)", skip = true },
]
```

Run:

```bash
git cliff --tag v1.4.2 --output CHANGELOG.md
git cliff --unreleased --strip header --output RELEASE_NOTES.md
```

### 4. `release-please` (GitHub Actions)

```yaml
# .github/workflows/release-please.yml
name: release-please
on:
  push: { branches: [main] }
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        with:
          release-type: simple
          package-name: awesome-mobile-app
```

`release-please` opens a PR that bumps the version and updates `CHANGELOG.md`; merging it creates a Git tag and GitHub Release.

### 5. Mapping to Store Release Notes

Stores have tight length limits:

- **Play**: 500 chars per locale per release.
- **App Store**: 4000 chars per locale per version.

Extract only `feat`/`fix`/`perf` lines of the latest release:

```bash
git cliff --unreleased \
  --strip all \
  | grep -E '^- (feat|fix|perf)' \
  | head -c 480 > store_notes.txt
```

Or keep a hand-curated `store_notes/en-US.txt` — automated notes are often too terse or too technical for end users.

### 6. iOS Store Metadata

Write per-locale under `ios/fastlane/metadata/<locale>/release_notes.txt`. `deliver` uploads them automatically.

### 7. Android Store Metadata

Write under `android/fastlane/metadata/android/<locale>/changelogs/<versionCode>.txt`. `supply` uploads them.

### 8. In-App "What's New"

Ship release notes as a local JSON bundled with the app:

```json
{
  "1.4.2": {
    "title": "April Update",
    "items": [
      "Faster checkout on slow networks",
      "Fixed crash opening PDF attachments"
    ]
  }
}
```

Compare `BuildConfig.VERSION_NAME` (or `CFBundleShortVersionString`) to last-seen version stored in prefs; show the matching entry on first launch.

### 9. CI Wiring

`.github/workflows/release.yml` (excerpt):

```yaml
- name: Generate release notes
  run: |
    git cliff --tag ${{ github.ref_name }} --output CHANGELOG.md
    git cliff --tag ${{ github.ref_name }} --strip all \
      | grep -E '^- (feat|fix|perf)' | head -c 480 > store_notes_en.txt

- name: Commit changelog
  run: |
    git config user.name "release-bot"
    git config user.email "bot@acme.com"
    git add CHANGELOG.md
    git commit -m "chore: update CHANGELOG for ${{ github.ref_name }}" || true
    git push
```

### 10. PR Template

Nudge contributors toward Conventional Commits with a PR template:

```markdown
## Summary
<!-- feat: / fix: / perf: / refactor: ... -->

## Why
<!-- link the ticket -->

## Risk
<!-- behind a flag? migration? -->
```

### 11. Anti-Patterns

- Writing notes by hand on release day — recency bias, missed fixes.
- Dumping raw commit messages into store notes — users don't care about `chore(deps): bump foo`.
- Using release notes as the changelog and nothing else — keep the full `CHANGELOG.md` for developers.
- Inconsistent wording between Play and App Store notes.

### 12. Checklist
- [ ] Conventional Commits enforced via `commitlint` + commit-msg hook.
- [ ] A generator (`git-cliff` / `release-please`) wired into CI.
- [ ] `CHANGELOG.md` auto-updated on release tags.
- [ ] Per-locale store notes generated or curated under `fastlane/metadata`.
- [ ] In-app What's New screen reads from a versioned JSON.
- [ ] Release notes identical in spirit across Play, App Store, and in-app.
