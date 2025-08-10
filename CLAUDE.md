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
4. **Shared Actions** (`setup-artifact`): Reusable composite actions for artifact management

### Workflow Dependencies
```
build-test-publish.yml (orchestrator)
├── test-and-build.yml (always runs)
├── publish-docker-image.yml (conditional)
├── publish-npm-libraries.yml (conditional)
├── publish-python-libraries.yml (conditional)
├── publish-firefox-extension.yml (conditional)
├── release-android-apk.yml (conditional)
└── release-github.yml (conditional)
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
- Implements comprehensive caching for dependencies across all tools
- Supports Nx monorepos with SHA optimization
- Handles Playwright E2E testing automatically
- Creates build artifacts for downstream workflows

### Publishing Workflows
Each specialized for different targets:
- **Docker**: Multi-platform builds (amd64/arm64), registry flexibility
- **npm**: Version comparison, multi-library support, input sanitization
- **Python**: UV-based publishing to PyPI
- **Firefox**: XPI packaging and AMO publishing
- **Android**: APK building with keystore management
- **GitHub**: Release creation with artifact attachment

## Working with This Repository

### Testing Workflow Changes
Since this repository contains reusable workflows, testing requires:
1. Create a test repository that references the workflows
2. Use the `@main` or specific branch/tag when referencing workflows
3. Test with minimal example projects for each supported tool

### Common Development Commands
This repository doesn't contain traditional build commands since it's pure GitHub Actions YAML. Instead:

```bash
# Validate YAML syntax
yamllint .github/workflows/

# Test workflow locally (if using act)
act -W .github/workflows/test-and-build.yml

# Check workflow references
grep -r "uses.*/.github/workflows" .github/workflows/
```

### Security Considerations
- All workflows implement input validation and sanitization
- Secrets are conditionally used (workflows only run when secrets exist)
- Timeouts prevent runaway builds (15-60 minutes depending on complexity)
- Minimal permissions principle applied to all jobs

### Workflow Input Patterns
Key input parameters across workflows:
- `tool`: Determines build system (npm, yarn, uv, ./gradlew, mvn, bash)
- `artifact_path`: Where build outputs are stored/retrieved
- `event_name`: Controls conditional execution (push vs pull_request)
- Platform-specific metadata (docker_meta, addon_guid, etc.)

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
Central pattern using `setup-artifact` action:
1. Check if build artifact exists from previous job
2. Download artifact if available and path specified
3. Conditional execution of publishing steps based on artifact existence

This ensures publishing workflows only run when there are actual build outputs to publish.