# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a collection of reusable GitHub Actions workflows designed to provide comprehensive CI/CD automation for multiple languages and platforms. The workflows support Node.js, Python, Java, Gradle, Maven, and Bash projects with automated testing, building, and publishing to various registries (Docker, npm, PyPI, Firefox and Thunderbird add-ons, Android APK releases).

## Architecture

### Core Workflow Pattern
The repository follows a hierarchical workflow structure:

1. **Universal Orchestrator** (`build-test-publish.yml`): Main entry point that orchestrates all other workflows based on inputs
2. **Core Build Workflow** (`test-and-build.yml`): Handles testing, building, and artifact creation
3. **Publishing Workflows**: Specialized workflows for different publication targets
4. **Third-party Actions**: External actions for artifact existence checking and downloading

### Workflow Dependencies
```
build-test-publish.yml (orchestrator)
├── lint.yml (validates workflows)
├── security-scan-source.yml (pre-build security scanning)
├── test-and-build.yml (always runs)
├── security-scan-artifacts.yml (post-build filesystem/artifact scanning)
├── publish-docker-image.yml (conditional)
├── publish-npm-libraries.yml (conditional)
├── publish-python-libraries.yml (conditional)
├── publish-firefox-extension.yml (conditional)
├── release-android-apk.yml (conditional)
├── release-github.yml (conditional)
├── post-publish-verification.yml (verifies published Docker images)
└── summarize-workflow.yml (always runs after all jobs)
```

## Key Workflows

### Universal Workflow (`build-test-publish.yml`)
The main orchestrator that:
- Takes comprehensive inputs for all supported tools and platforms
- Conditionally triggers publishing workflows based on `github.event_name` (push vs pull_request) and input parameters
- Supports multi-tool builds (npm, yarn, uv, cargo, ./gradlew, mvn, bash)

### Test and Build (`test-and-build.yml`)
Core workflow that:
- Sets up language-specific environments (Node.js v24.20.0, Python via pyproject.toml, Java 21)
- Implements comprehensive caching for dependencies across all tools (tool-specific cache keys)
- Supports Nx monorepos with SHA optimization
- Handles Playwright E2E testing automatically (supports .ts, .js, and .mjs config variants)
- Creates build artifacts for downstream workflows with descriptive suffixes

### Publishing Workflows
Each specialized for different targets:
- **Docker**: Multi-platform builds (amd64/arm64), registry flexibility, fail-fast: false for matrix builds
- **npm**: Version comparison, multi-library support, input sanitization, dry-run validation, **uses OIDC Trusted Publishing (no NPM_TOKEN required)**, **SBOM generation and attestation with Sigstore**
- **Python**: artifact validation + version tag; the direct `uv publish` step is **parked** (PyPI Trusted Publishing does not work through a reusable workflow) — publish from a tag-triggered workflow in the calling repo
- **Firefox/Thunderbird**: XPI packaging, publishing to AMO or ATN via `addon_api_url_prefix`; `addon_approval_timeout` defaults to `0` because AMO never auto-signs listed add-ons, so there is no signed XPI to wait for
- **Android**: APK building with keystore management
- **GitHub**: Release creation with artifact attachment, supports `overwrite_release` for non-semver workflows

#### NPM SBOM Attestation
The npm publishing workflow generates and attests Software Bill of Materials (SBOM) for supply chain security:
- **Automatic SBOM generation**: Creates SBOM from package-lock.json/yarn.lock using CycloneDX
- **Sigstore attestation**: Signs SBOM with keyless signing via GitHub's OIDC (eliminates need for signing keys)
- **Format**: CycloneDX (`sbom/sbom.cyclonedx.json`)
- **Artifact retention**: SBOM uploaded as workflow artifact with 90-day retention
- **Verification**: Consumers can verify attestations using `npm audit signatures`

**Configuration:** SBOM attestation is controlled by `enable_sbom_attestation`, an
input of `test-and-build.yml` (default: `true`). It is **not exposed by the
`build-test-publish.yml` orchestrator** — callers using the orchestrator get the
default and cannot turn it off; to configure it, call `test-and-build.yml`
directly. It only takes effect for `tool: npm` or `tool: yarn`.

The SBOM is generated in **CycloneDX** format (`sbom/sbom.cyclonedx.json`). There
is no `sbom_format` input.

**Verifying SBOM attestations as a consumer:**
```bash
# Download attestation bundle for a published package
npm audit signatures <package-name>

# View SBOM details
gh attestation verify oci://registry.npmjs.org/<namespace>/<package>@<version> \
  --owner <github-org>
```

**Benefits:**
- ✅ Supply chain transparency: Full visibility into all dependencies
- ✅ Vulnerability tracking: Quick querying against known malicious packages
- ✅ Compliance: Meet SLSA/SSDF regulatory requirements
- ✅ Incident response: Rapid impact analysis during supply chain attacks

### Security Scanning Workflows
**Triple-layer defense-in-depth security approach** with 100% free, open-source tools:

#### Layer 1: Pre-Build Security Scan (`security-scan-source.yml`)
Scans source code and dependencies **before building**:
- **Semgrep**: Fast SAST for all languages (configurable rulesets)
- **Bandit**: Python-specific source code security analysis
- **pip-audit**: Python dependency vulnerability scanning (official PyPA tool)
- **npm audit** / **yarn audit**: Node.js dependency vulnerability scanning
- Uploads SARIF reports to GitHub Security tab
- Fails fast to prevent building vulnerable code

##### npm audit auto-fix (`npm-audit-autofix.yml`)
When `npm audit` fails, this workflow runs `npm audit fix` and opens a PR with the
resulting `package-lock.json` changes. It has two modes, both dispatched
automatically from `security-scan-source.yml`:

| | `branch` | `scheduled` |
|---|---|---|
| Trigger | any PR branch in this repo (Dependabot **and** feature branches) | `schedule` or `workflow_dispatch` |
| PR base | the triggering branch | default branch (or `audit_fix_base_branch`) |
| Fix branch | `audit-fix/<branch>-<sha>` (one per push) | `audit-fix/scheduled` (reused) |
| Existing PR | new PR each time | updated in place, branch is force-pushed |

The `branch` mode covers everyday work: push to a feature branch, `npm audit`
fails, and a fix PR is opened against *your* branch. Merge it and carry on — the
fix travels into `main` with your own PR. The PR body adapts to whether the
trigger was Dependabot or a regular branch.

The `scheduled` mode is the low-intervention path: the daily scan finds a new
advisory, fixes it, and leaves a single mergeable PR against `main`. Because the
fix branch is rebuilt from base on every run, later runs force-push it and edit
the same PR instead of piling up duplicates.

**Fork PRs are skipped.** A `pull_request` from a fork gets a read-only
`GITHUB_TOKEN`, so the push could not succeed; the job is skipped via a
`head.repo.full_name == github.repository` guard rather than failing. Fixing
those requires `pull_request_target`, which runs untrusted PR code with write
permissions — deliberately not done here.

Both modes run `npm audit fix` **without** `--force`, so only semver-compatible
updates are applied and `package.json` is never touched. Findings that need a
major upgrade stay open and are called out in the PR body.

**Configuration:**
```yaml
inputs:
  enable_npm_audit_autofix: true  # Enable/disable both modes (default: enabled)
  audit_fix_base_branch: ""       # Base for scheduled PRs (default: repo default branch)
```

Calling workflows need `contents: write` and `pull-requests: write` for the
auto-fix job to push the branch and open the PR.

#### Layer 2: Post-Build Artifact Scan (`security-scan-artifacts.yml`)
Scans **build artifacts** before publishing:
- **Trivy**: Comprehensive filesystem scanner for packages and dependencies
- **Grype**: Alternative vulnerability scanner for redundancy
- Scans filesystem artifacts from `artifact_path`
- Generates security summary tables in workflow output
- Acts as security gate before publishing

#### Layer 3: Post-Publish Verification (`post-publish-verification.yml`)
Verifies **published Docker images** after deployment:
- **Trivy**: Scans published Docker images pulled from registry
- Authenticates to GHCR using GitHub OIDC token
- Runs after `publish_docker_image` job completes
- Scans actual published images to detect post-build supply chain attacks
- Uploads SARIF reports to GitHub Security tab
- Only runs when `docker_meta` is provided and publishing is enabled

**Configuration:**
```yaml
inputs:
  enable_security_scanning: true  # Enable/disable (default: enabled)
  semgrep_rules: "auto"              # auto, p/security-audit, p/owasp-top-ten, p/ci
  trivy_severity: "MEDIUM,HIGH,CRITICAL"  # Severity threshold
  trivy_exit_code: 1               # 0=warn only, 1=fail build
```

**All security tools are:**
- ✅ Completely free for open source (no signup, no limits)
- ✅ Open source projects with active maintenance
- ✅ Industry-standard tools used by major projects

### Summary Workflow (`summarize-workflow.yml`)
Dedicated workflow for result aggregation that:
- Collects status from all workflows including security scans
- Generates comprehensive workflow summary with status table
- Tracks and outputs published artifacts list
- Provides visual status indicators (✅ Published, ⏭️ Skipped, 🔒 Security Passed)
- Recently refactored from 90 lines to 30 lines (67% reduction) using helper functions

## Working with This Repository

### Testing Workflow Changes
Since this repository contains reusable workflows, testing requires:
1. Create a test repository that references the workflows
2. Use the `@main` or specific branch/tag when referencing workflows
3. Test with minimal example projects for each supported tool

### Common Development Commands
This repository doesn't contain traditional build commands since it's pure GitHub Actions YAML. Instead:

```bash
# Lint workflows with actionlint (recommended)
actionlint .github/workflows/*.yml

# Validate YAML syntax
yamllint .github/workflows/

# Test workflow locally (if using act)
act -W .github/workflows/test-and-build.yml

# Check workflow references
grep -r "uses.*/.github/workflows" .github/workflows/
```

### Workflow Validation with actionlint
The repository uses [actionlint](https://github.com/rhysd/actionlint) for static analysis of workflow files:
- **Pre-execution validation gate**: The `lint.yml` workflow is called as the first job in `build-test-publish.yml`
- **Safety mechanism**: All jobs depend on successful lint validation - invalid workflows cannot execute
- **Automated CI**: Also runs independently on every push/PR affecting workflow files
- **Local validation**: Run `actionlint .github/workflows/*.yml` before committing
- **Reusable workflow support**: actionlint validates inputs/outputs/secrets in reusable workflows
- **Installation**: `bash <(curl https://raw.githubusercontent.com/rhysd/actionlint/main/scripts/download-actionlint.bash)`

#### Validation Gate Architecture
```
build-test-publish.yml execution flow:
  1. lint (validates all workflows) ← MUST PASS
      ├─ Generate hash of all workflow files
      ├─ Check cache for this hash
      ├─ If cache hit: skip validation (instant ✅)
      └─ If cache miss: run actionlint + cache result
  2. security_scan_source (pre-build security) ← MUST PASS
      ├─ Semgrep SAST (all languages)
      ├─ Bandit (Python source code)
      ├─ pip-audit (Python dependencies)
      └─ npm/yarn audit (Node.js dependencies)
  3. test_and_build (needs: security_scan_source) ← Only runs if security passes
  4. security_scan_artifacts (pre-publish security) ← MUST PASS
      ├─ Trivy (filesystem artifacts)
      └─ Grype (backup filesystem scanner)
  5. [publishing jobs] (needs: security_scan_artifacts) ← Only runs if artifacts are secure
      ├─ publish_docker_image
      ├─ publish_npm_libraries
      ├─ publish_python_libraries
      ├─ publish_firefox_extension
      ├─ release_android_apk
      ├─ release_github
      └─ publish_crates_io
  6. post_publish_verification (post-publish security) ← MUST PASS
      └─ Trivy (scans published Docker images from registry)
  7. summarize (needs: all jobs)
```

This ensures that workflows on the `main` branch are always valid and secure before execution, during publishing, and after deployment.

#### Cross-Repository Validation Caching
The lint workflow implements intelligent caching to avoid redundant validation:
- **Hash-based cache key**: Generates SHA256 hash of all workflow files
- **Cross-repository sharing**: Cache is shared across ALL repositories using these workflows
- **Automatic invalidation**: Cache key changes when any workflow file is modified
- **Performance**: Subsequent runs skip validation if workflows haven't changed (1-2 second overhead vs 30+ seconds)

**Example flow:**
1. Repository A runs workflow → cache miss → validates workflows → caches result with hash `abc123`
2. Repository B runs workflow (same workflow version) → cache hit for `abc123` → skips validation ✅
3. Workflow updated in main → hash becomes `def456` → next run is cache miss → validates → caches new result
4. All repositories using updated workflows → cache hit for `def456` → skip validation ✅

This dramatically reduces validation overhead while maintaining safety guarantees.

### Security Considerations
- All workflows implement input validation and sanitization
- Enhanced security with minimal permissions (contents: read by default)
- Early secret validation with categorized exit codes (2: missing secrets, 3: invalid input)
- Secrets are conditionally used (workflows only run when secrets exist)
- Optimized timeouts prevent runaway builds (5-60 minutes depending on complexity)
- Minimal permissions principle applied to all jobs

### Required Permissions for Calling Workflows
**IMPORTANT:** All calling workflows **MUST** include these permissions:

```yaml
jobs:
  build_and_deploy:
    uses: tehw0lf/workflows/.github/workflows/build-test-publish.yml@main
    permissions:
      id-token: write       # REQUIRED - Always needed for OIDC (npm Trusted Publishing + future integrations)
      attestations: write   # Required for SBOM provenance attestation (npm/yarn builds)
      actions: write        # Required for workflow management
      contents: write       # Required for GitHub releases
      packages: write       # Required for Docker/GHCR publishing
      security-events: write # Required for SARIF uploads (security scanning)
    with:
      # ... inputs
```

**Why is `id-token: write` always required?**
- Currently used for npm and Python Trusted Publishing (eliminates need for NPM_TOKEN and UV_TOKEN secrets)
- Planned for future OIDC integrations with other publishing targets (Docker registries, etc.)
- Due to GitHub Actions limitations, permissions cannot be conditionally granted in reusable workflows
- Must be set at the top-level calling workflow, even if not publishing to npm or PyPI
- Cannot be controlled with `if` conditions - permissions are evaluated before job execution

**Why is `attestations: write` required?**
- The `test_and_build` job attests SBOM provenance with Sigstore and needs it
- Only active for `tool: npm` / `tool: yarn`, but must be granted regardless —
  permissions are evaluated before the job runs and cannot be conditional
- Omitting it fails the build at the attestation step

**Why is `security-events: write` required?**
- Enables SARIF uploads to GitHub Security tab for code scanning alerts
- Provides centralized security vulnerability tracking across repositories
- Required for Semgrep, Bandit, Trivy, and Grype security reports

### Workflow Input Patterns

**The `workflow_call.inputs` block of `build-test-publish.yml` is the single source of
truth** for the 44 available inputs — read it rather than trusting any list, here or in
the README. The README's "Optional Inputs (complete)" table mirrors it and must be
updated in the same commit whenever an input is added, renamed or removed.

Key parameters:
- `tool`: Determines build system (`npm`, `yarn`, `uv`, `cargo`, `./gradlew`, `mvn`, `bash`)
- `root_dir`: Project root — required for monorepos, defaults to `.`
- `artifact_path`: Where build outputs are stored/retrieved
- `enable_security_scanning`: Enable/disable security scanning (default: `true`)
- `semgrep_rules`: Semgrep ruleset configuration (default: "auto")
- `trivy_severity`: Minimum severity threshold (default: "MEDIUM,HIGH,CRITICAL")
- `trivy_exit_code`: Fail build on vulnerabilities (default: `1`)
- Platform-specific metadata (`docker_meta`, `xpi_path`, etc.)

**Complete input list** (44; mirror of `workflow_call.inputs` — verify against the
YAML before relying on it):

| Group | Inputs |
|---|---|
| Project | `tool`, `root_dir`, `head_ref`, `runner` |
| Scripts | `install`, `format`, `lint`, `test`, `e2e`, `build_branch`, `build_main`, `post_build_script` |
| Artifacts | `artifact_path` |
| Docker | `docker_meta`, `docker_namespace`, `registry`, `platforms`, `docker_pre` |
| npm | `libraries`, `library_path`, `npm_namespace`, `cyclonedx_ignore_npm_errors` |
| Python | `publish_python_libraries` |
| Rust | `rust_version`, `enable_clippy`, `enable_rustfmt`, `clippy_args`, `cargo_features`, `cargo_dry_run`, `cargo_package_name`, `cargo_publish_flags` |
| Extensions | `xpi_path`, `addon_api_url_prefix`, `addon_channel`, `addon_approval_timeout` |
| Android | `app_root` |
| Release | `publish_github_release`, `release_pre` |
| Security | `enable_security_scanning`, `semgrep_rules`, `npm_audit_omit_dev`, `npm_audit_severity_threshold`, `trivy_severity`, `trivy_exit_code` |

Script inputs are appended to `tool`, so the value is the subcommand only
(`build_main: "run build"`, not `"npm run build"`). Defaults are documented in the
README's input table.

**There is no `event_name` input.** Conditional execution reads `github.event_name`
directly inside the workflow. Passing `event_name:` from a caller is an error — GitHub
rejects unknown inputs to a reusable workflow.

**Input types are enforced; there is no coercion.** A quoted `"true"` passed to a
`type: boolean` input fails the whole run with `startup_failure` before any job
starts — it does not evaluate as truthy, and no step appears in the log. Same for
a quoted number against a `type: number` input. This is verified behaviour, not
inference. `actionlint` does not catch it (it accepts wrong types and even
unknown input names), so the lint gate offers no protection here.

Booleans: `publish_github_release`, `publish_python_libraries`,
`enable_security_scanning`, `npm_audit_omit_dev`, `enable_clippy`,
`enable_rustfmt`, `cargo_dry_run`, `cyclonedx_ignore_npm_errors`, and
`enable_sbom_attestation` / `enable_npm_audit_autofix` / `fail_on_warn` in the
sub-workflows. Numbers: `trivy_exit_code`, `max_duration_minutes`.

When changing an input's type, **update every caller repo first** — unquoted
values are valid under both `string` and the stricter type, so callers can land
ahead of the type change with no breaking window.

**Inputs that gate a job.** Publishing jobs require a `push` event *plus* their own
non-empty input. Missing the second half is the usual cause of "the job silently did
not run":

| Job | Gated on |
|---|---|
| Docker | `docker_meta` |
| npm | `library_path` (**not** `libraries`) |
| PyPI (tag only, upload parked) | `tool: uv` **and** `publish_python_libraries: true` |
| Firefox / Thunderbird | `xpi_path` |
| Android | `app_root` |
| GitHub release | `artifact_path` **and** `publish_github_release: true` |
| crates.io | `tool: cargo` |

Setting `libraries` without `library_path` publishes nothing at all.

#### GitHub Release Versioning
Releases use automatic version extraction from the project manifest — no `release_tag` input needed:
- **npm/yarn**: reads `version` from `package.json` via `jq`
- **uv**: reads `version` from `pyproject.toml` (falls back to `version.json` in artifact path)
- **cargo**: reads `version` from `Cargo.toml` via `grep`
- **other tools**: not supported for GitHub releases (fails with clear error)

The pipeline tags `vX.Y.Z` and creates the release. If the tag already exists, the workflow fails — bump the version in the manifest to create a new release.

**Example usage:**
```yaml
uses: ./.github/workflows/build-test-publish.yml
with:
  publish_github_release: true
  # No release_tag needed — version is read from package.json / pyproject.toml
```

### Publishing Triggers
Publishing only occurs on:
- `push` events (typically main branch)
- When required secrets are available
- When relevant input parameters are provided
- When build artifacts exist from prior jobs

### Multi-language Support
The workflows dynamically adapt based on `tool` parameter:
- **npm/yarn**: Node.js v24.20.0, package-lock.json/yarn.lock caching
- **uv**: Python setup from pyproject.toml, uv.lock caching
- **./gradlew**: JDK 21 Temurin, Gradle caching
- **mvn**: JDK 21 Temurin, Maven repository caching
- **bash**: Shell script execution with basic environment setup

### Artifact Management
Consistent pattern across all publishing workflows:
1. Check if build artifact exists using `softwareforgood/check-artifact-v4-existence@v0`
2. Download artifact if available using `actions/download-artifact@v4`
3. Conditional execution of publishing steps based on artifact existence

This ensures publishing workflows only run when there are actual build outputs to publish.

### Automated Maintenance
Dependency updates are split between two bots with **no overlap**, so the same
change is never proposed twice:

| Bot | Scope | Config |
|---|---|---|
| Dependabot | `uses:` action references (SHA pins) | `.github/dependabot.yml` |
| Renovate | version pins **inside** workflow steps | `renovate.json` |

**Dependabot** handles GitHub Actions versions on a weekly schedule.

**Renovate** covers what Dependabot structurally cannot see: versions that live
in `run:` commands, `with:` inputs and workflow input defaults. Without it these
pins silently rot — Trivy was once 5.5 months out of date while all 22 actions
stayed current, because nothing was watching it.

`renovate.json` sets `enabledManagers: ["custom.regex"]`, which is what keeps
Renovate off the `uses:` lines that Dependabot owns. **Do not add
`github-actions` to `enabledManagers`** unless Dependabot is removed first.

Renovate's custom managers track:
- **Trivy** — the `install.sh` version, the matching `actions/cache` key, and
  the `version:` input of `aquasecurity/trivy-action`. All three are grouped
  into one PR so every scan job keeps running the same engine version.
- **semgrep / bandit** — `pip install <tool>==<version>`, grouped together.
  Extras such as `bandit[toml]` are preserved on update.
- **Node.js** — `node-version:` / `node_version:` inputs and the
  `npm-audit-autofix.yml` default. Major updates are disabled so the pin stays
  on the active LTS line rather than jumping to an odd/current release.
- **npm CLI** — the `npm install -g npm@<major>` pin; minor/patch updates are
  disabled since the pin only expresses a major.
- **Java** — the `java-version:` pins in `setup-java` steps. The two pins differ
  on purpose (21 in `test-and-build.yml`, 24 in `release-android-apk.yml`), so
  major updates are disabled; changing a major is a deliberate, tested change.

**When adding a new pinned tool version to a workflow, add a matching custom
manager** — otherwise the pin is invisible to both bots and will go stale.

### Recent Optimizations (Phase 1-8)
Key improvements made to the workflow suite:
1. **Artifact clarity**: Added descriptive suffixes to artifact uploads
2. **Output cleanup**: Removed unused workflow outputs
3. **Validation**: Added explicit artifact validation in Python workflow
4. **Resilience**: Added fail-fast: false to Docker matrix builds
5. **Playwright support**: Extended config detection to .ts, .js, and .mjs variants
6. **Code reduction**: Refactored summary workflow (67% line reduction)
7. **Automation**: Added Dependabot for weekly action updates
8. **OIDC Integration**: Migrated npm and Python publishing to Trusted Publishing (eliminates NPM_TOKEN and UV_TOKEN secret requirements)
9. **Security Scanning**: Implemented triple-layer defense-in-depth security with free open-source tools (Semgrep, Bandit, pip-audit, npm audit, Trivy, Grype)
10. **Post-Publish Verification**: Added dedicated workflow to scan published Docker images from registry (prevents timing issues and authentication failures)
11. **Automatic release versioning**: Removed `release_tag` and `overwrite_release` inputs; version is now read from the manifest (package.json / pyproject.toml) and used as the git tag — forces proper semver and auto-generates release notes

### Known Correct Patterns (Do Not Change)
These patterns are intentionally designed and verified as correct:
- Docker/Python workflows use artifact path without root_dir prefix
- NPM workflow uses root_dir prefix (required for Nx monorepos)
- Tool-specific caching already optimized with ${{ inputs.tool }} in cache keys
- npm dry-run serves validation purpose (catches errors before publish)
- All checkout steps are necessary for their specific purposes