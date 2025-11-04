# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a collection of reusable GitHub Actions workflows designed to provide comprehensive CI/CD automation for multiple languages and platforms. The workflows support Node.js, Python, Java, Gradle, Maven, and Bash projects with automated testing, building, and publishing to various registries (Docker, npm, PyPI, Firefox Add-ons, Android APK releases).

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
├── test-and-build.yml (always runs)
├── publish-docker-image.yml (conditional)
├── publish-npm-libraries.yml (conditional)
├── publish-python-libraries.yml (conditional)
├── publish-firefox-extension.yml (conditional)
├── release-android-apk.yml (conditional)
├── release-github.yml (conditional)
└── summarize-workflow.yml (always runs after publishing)
```

## Key Workflows

### Universal Workflow (`build-test-publish.yml`)
The main orchestrator that:
- Takes comprehensive inputs for all supported tools and platforms
- Conditionally triggers publishing workflows based on `event_name` and input parameters
- Supports multi-tool builds (npm, yarn, uv, ./gradlew, mvn, bash)

### Test and Build (`test-and-build.yml`)
Core workflow that:
- Sets up language-specific environments (Node.js 20, Python via pyproject.toml, Java 21)
- Implements comprehensive caching for dependencies across all tools (tool-specific cache keys)
- Supports Nx monorepos with SHA optimization
- Handles Playwright E2E testing automatically (supports .ts, .js, and .mjs config variants)
- Creates build artifacts for downstream workflows with descriptive suffixes

### Publishing Workflows
Each specialized for different targets:
- **Docker**: Multi-platform builds (amd64/arm64), registry flexibility, fail-fast: false for matrix builds
- **npm**: Version comparison, multi-library support, input sanitization, dry-run validation
- **Python**: UV-based publishing to PyPI with explicit artifact validation
- **Firefox**: XPI packaging and AMO publishing
- **Android**: APK building with keystore management
- **GitHub**: Release creation with artifact attachment, supports `overwrite_release` for non-semver workflows

### Summary Workflow (`summarize-workflow.yml`)
Dedicated workflow for result aggregation that:
- Collects status from all publishing workflows
- Generates comprehensive workflow summary with status table
- Tracks and outputs published artifacts list
- Provides visual status indicators (✅ Published, ⏭️ Skipped)
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
  2. test_and_build (needs: lint) ← Only runs if lint passes
  3. [publishing jobs] (needs: test_and_build)
  4. summarize (needs: all publishing jobs)
```

This ensures that workflows on the `main` branch are always valid before execution.

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

### Workflow Input Patterns
Key input parameters across workflows:
- `tool`: Determines build system (npm, yarn, uv, ./gradlew, mvn, bash)
- `artifact_path`: Where build outputs are stored/retrieved
- `event_name`: Controls conditional execution (push vs pull_request)
- `overwrite_release`: Enables non-semver workflows by deleting and recreating releases (default: false)
- Platform-specific metadata (docker_meta, addon_guid, etc.)

#### Release Overwrite Feature
The `overwrite_release` parameter enables non-semver release workflows:
- **Use case**: Projects using fixed tags like `latest`, `v1.0.0`, or date-based tags
- **Behavior**: When `true`, deletes existing release and tag before creating new one
- **Default**: `false` (prevents accidental overwrites, fails with error if release exists)
- **Security**: Requires `contents: write` permission (already granted in release workflow)
- **Example usage**:
  ```yaml
  uses: ./.github/workflows/build-test-publish.yml
  with:
    publish_github_release: "true"
    release_tag: "latest"
    overwrite_release: "true"  # Enables fixed-tag releases
  ```

### Publishing Triggers
Publishing only occurs on:
- `push` events (typically main branch)
- When required secrets are available
- When relevant input parameters are provided
- When build artifacts exist from prior jobs

### Multi-language Support
The workflows dynamically adapt based on `tool` parameter:
- **npm/yarn**: Node.js 20, package-lock.json/yarn.lock caching
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
The repository includes Dependabot configuration (`.github/dependabot.yml`) for:
- Weekly automated updates to GitHub Actions versions
- Ensures security patches are applied promptly
- Reduces manual maintenance burden

### Recent Optimizations (Phase 1-3)
Key improvements made to the workflow suite:
1. **Artifact clarity**: Added descriptive suffixes to artifact uploads
2. **Output cleanup**: Removed unused workflow outputs
3. **Validation**: Added explicit artifact validation in Python workflow
4. **Resilience**: Added fail-fast: false to Docker matrix builds
5. **Playwright support**: Extended config detection to .ts, .js, and .mjs variants
6. **Code reduction**: Refactored summary workflow (67% line reduction)
7. **Automation**: Added Dependabot for weekly action updates

### Known Correct Patterns (Do Not Change)
These patterns are intentionally designed and verified as correct:
- Docker/Python workflows use artifact path without root_dir prefix
- NPM workflow uses root_dir prefix (required for Nx monorepos)
- Tool-specific caching already optimized with ${{ inputs.tool }} in cache keys
- npm dry-run serves validation purpose (catches errors before publish)
- All checkout steps are necessary for their specific purposes