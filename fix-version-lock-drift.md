# Fix pyproject.toml Version / uv.lock Drift

## Problem

The current `auto-tag.yml` workflow derives the release version from the latest git tag + commit message prefix (conventional commits). But `pyproject.toml` contains a hardcoded `version` field, and `uv.lock` records the root package version too. These three sources of truth drift apart:

1. PR author manually bumps `pyproject.toml` (guessing the next version)
2. `auto-tag.yml` computes the version independently from the latest tag
3. `uv.lock` still has the old version until someone re-locks

This has already caused mismatches (e.g. PR #84 set `1.17.12` in pyproject.toml but auto-tag created `v1.18.0`), requiring manual hotfix commits.

**Affected repos:** legato-common (and any other repo using the shared `auto-tag.yml` from `legatoai/.github`)

## Option A: python-semantic-release (PSR)

**Replaces** the current custom `auto-tag.yml` with PSR's GitHub Action.

### How it works

1. PR merges to main with conventional commit prefix (`feat:`, `fix:`, `feat!:`)
2. PSR determines the bump type (same logic as current auto-tag)
3. PSR bumps `pyproject.toml` version
4. PSR runs a configurable `build_command` that re-locks and stages:
   ```toml
   [tool.semantic_release]
   build_command = """
   uv lock --upgrade-package "$PACKAGE_NAME"
   git add uv.lock
   uv build
   """
   ```
5. PSR creates **one commit** with: bumped `pyproject.toml` + updated `uv.lock` + CHANGELOG.md
6. PSR creates the git tag on that commit, pushes both
7. PSR creates a GitHub Release

### Loop prevention

- `GITHUB_TOKEN` pushes don't trigger new workflow runs (GitHub built-in rule)
- Optionally add `[skip ci]` to the release commit message:
  ```toml
  [tool.semantic_release]
  commit_message = "chore(release): {version} [skip ci]"
  ```

### Pros
- Drop-in replacement for auto-tag — same conventional commit convention
- Proper `uv lock --upgrade-package` (only bumps own version entry, no dependency churn)
- Single source of truth: PSR owns the version, no human guessing
- Generates CHANGELOG.md automatically
- Well-established in the Python ecosystem

### Cons
- New dependency (PSR GitHub Action + pyproject.toml config)
- Need to update all repos using auto-tag.yml
- `build_command` needs uv installed in CI (PSR runs in Docker by default)

### Config example

```toml
# pyproject.toml
[tool.semantic_release]
version_toml = ["pyproject.toml:project.version"]
commit_message = "chore(release): {version}"
build_command = """
uv lock --upgrade-package "$PACKAGE_NAME"
git add uv.lock
uv build
"""

[tool.semantic_release.branches.main]
match = "main"
```

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    branches: [main]
jobs:
  release:
    runs-on: ubuntu-24.04-arm
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          fetch-tags: true
      - uses: astral-sh/setup-uv@v6
      - uses: python-semantic-release/python-semantic-release@v9
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

---

## Option B: Dynamic Versioning via hatch-vcs

**Eliminates** the hardcoded version entirely. Version is derived from git tags at build time.

### How it works

1. Remove `version = "x.y.z"` from `pyproject.toml`, replace with `dynamic = ["version"]`
2. Add `hatch-vcs` as build dependency — it reads version from `git describe`
3. `auto-tag.yml` (or PSR) creates the tag as before
4. Any `uv build` / `pip install` resolves the version from the nearest tag
5. `uv.lock` records `version = "0.0.0"` or similar placeholder for editable/source installs (no drift)

### Config example

```toml
# pyproject.toml
[project]
name = "legato-common"
dynamic = ["version"]

[build-system]
requires = ["hatchling", "hatch-vcs"]
build-backend = "hatchling.build"

[tool.hatch.version]
source = "vcs"

[tool.hatch.build.hooks.vcs]
version-file = "src/legato_common/_version.py"
```

### Pros
- Zero version management — no field to bump, no lock to update
- No drift possible — single source of truth (git tags)
- Works with the existing `auto-tag.yml` (it just creates tags)
- No new CI workflow needed

### Cons
- Bigger refactor: must update `pyproject.toml`, build config, and any code that reads `__version__`
- Consumers pinning by git tag still work, but editable installs show dev versions like `1.18.0.dev3+g6aed426`
- Need to verify uv.lock behavior with dynamic versions — may still record a version that drifts
- All repos need the change

---

## Recommendation

**Option A (PSR)** is the safer incremental step — it fixes the drift with minimal disruption to the existing workflow. The conventional commit convention is already in place, and PSR is a drop-in for `auto-tag.yml`.

**Option B (hatch-vcs)** is cleaner long-term but higher risk/effort. Consider it as a follow-up after PSR is stable.

## Scope

- [ ] Prototype on legato-common first
- [ ] Update `legatoai/.github` shared workflows if replacing auto-tag
- [ ] Roll out to code-agency and other repos using auto-tag
- [ ] Verify consumer repos (code-agency) can still pin by tag in `[tool.uv.sources]`
