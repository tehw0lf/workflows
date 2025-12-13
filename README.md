# GitHub Actions Workflows Documentation

This repository contains a comprehensive set of reusable GitHub Actions workflows for building, testing, and publishing various types of applications and libraries.

## 🚀 Quick Start

To use these workflows in your project, create a workflow file in your repository's `.github/workflows/` directory:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build_and_deploy:
    uses: tehw0lf/workflows/.github/workflows/build-test-publish.yml@main
    permissions:
      actions: write     # Required for workflow management
      contents: write    # Required for GitHub releases
      packages: write    # Required for Docker/GHCR publishing
      id-token: write    # REQUIRED - Always needed (currently for npm Trusted Publishing, planned for future OIDC integrations)
    with:
      tool: npm
      lint: "run lint"
      test: "run test"
      build_main: "run build"
      artifact_path: "dist"
      event_name: ${{ github.event_name }}
      docker_meta: '[{"name":"my-app","file":"Dockerfile"}]'
      libraries: "lib1,lib2"
      library_path: "packages"
    secrets: inherit
```

**Important permissions:**
- **`id-token: write` is REQUIRED for all workflows** - Currently used for npm and Python Trusted Publishing (no NPM_TOKEN or UV_TOKEN needed!), with plans to extend OIDC to other publishing workflows in the future
- Due to GitHub Actions limitations, this permission must be set at the top-level calling workflow, regardless of which publishing workflows you use
- If using `release-it` for npm, add `.release-it.json` with `{"npm": {"skipChecks": true}}`

## 📋 Available Workflows

### 1. Universal Workflow (`build-test-publish.yml`)

The main orchestrator workflow that handles the complete CI/CD pipeline.

**Key Features:**
- ✅ Multi-language support (Node.js, Python, Java, Gradle, Bash)
- ✅ Automated testing and building
- ✅ Multi-platform publishing (Docker, npm, PyPI, Firefox, Android)
- ✅ Conditional deployment based on branch and inputs

**Required Inputs:**
- `event_name`: GitHub event name (required)

**Optional Inputs:**
- `tool`: Build tool (npm, yarn, uv, ./gradlew, mvn, bash)
- `lint`: Linting command
- `test`: Test command
- `build_main`: Build command for main branch
- `artifact_path`: Path to build artifacts
- `docker_meta`: Docker metadata JSON
- `libraries`: Comma-separated list of libraries to publish
- And many more...

### 2. Test and Build (`test-and-build.yml`)

Core workflow for testing and building applications.

**Features:**
- ✅ Advanced dependency caching for faster builds (tool-specific cache keys)
- ✅ Multi-language toolchain setup
- ✅ Nx monorepo support
- ✅ Playwright E2E testing (supports .ts, .js, and .mjs config variants)
- ✅ Descriptive artifact upload suffixes for clarity
- ✅ Configurable timeouts (45 minutes)

### 3. Docker Publishing (`publish-docker-image.yml`)

Publishes Docker images to container registries.

**Features:**
- ✅ Multi-platform builds (linux/amd64, linux/arm64)
- ✅ Flexible registry support (GHCR, Docker Hub, private registries)
- ✅ Input validation for security
- ✅ Matrix builds for multiple images with fail-fast: false
- ✅ Timeout protection (30 minutes)

### 4. NPM Libraries (`publish-npm-libraries.yml`)

Publishes Node.js libraries to npm registry using **Trusted Publishing** (Provenance).

**Features:**
- ✅ **Trusted Publishing**: No NPM_TOKEN required - uses OpenID Connect (OIDC)
- ✅ Version comparison to prevent duplicate publishes
- ✅ Multi-library support
- ✅ Security: Input sanitization and validation
- ✅ Dry-run validation (catches errors before publish)
- ✅ Timeout protection (20 minutes)

**Important:** Requires `id-token: write` permission instead of NPM_TOKEN secret

### 5. Python Libraries (`publish-python-libraries.yml`)

Publishes Python packages to PyPI using `uv` and **Trusted Publishing** (Provenance).

**Features:**
- ✅ **Trusted Publishing**: No UV_TOKEN required - uses OpenID Connect (OIDC)
- ✅ UV package manager support
- ✅ Automatic dependency management
- ✅ Explicit artifact validation with clear error messages
- ✅ Timeout protection (15 minutes)

**Important:** Requires `id-token: write` permission instead of UV_TOKEN secret

### 6. Firefox Extension (`publish-firefox-extension.yml`)

Publishes Firefox browser extensions to Mozilla Add-ons.

**Features:**
- ✅ Automated packaging (XPI creation)
- ✅ AMO (addons.mozilla.org) publishing
- ✅ Timeout protection (15 minutes)

### 7. Android APK (`release-android-apk.yml`)

Builds and releases Android APK files.

**Features:**
- ✅ Automated keystore generation and caching
- ✅ APK signing and alignment
- ✅ GitHub releases integration
- ✅ Timeout protection (30 minutes)

### 8. GitHub Releases (`release-github.yml`)

Creates GitHub releases with artifacts.

**Features:**
- ✅ Automatic version detection (Python projects)
- ✅ Configurable release tags
- ✅ Artifact attachment
- ✅ Timeout protection (10 minutes)

### 9. Workflow Summary (`summarize-workflow.yml`)

Aggregates and reports results from all publishing workflows.

**Features:**
- ✅ Comprehensive status tracking across all workflows
- ✅ Visual summary table with status indicators
- ✅ Published artifacts tracking and output
- ✅ Refactored from 90 lines to 30 lines (67% reduction) using helper functions
- ✅ Quick timeout (5 minutes)

## 🔧 Setup Instructions

### 1. Required Secrets & Permissions

#### Required Permissions (ALL WORKFLOWS)

**IMPORTANT:** The `id-token: write` permission is **REQUIRED for all workflows**, regardless of which publishing targets you use:

```yaml
jobs:
  build_and_deploy:
    uses: tehw0lf/workflows/.github/workflows/build-test-publish.yml@main
    permissions:
      id-token: write    # REQUIRED - Always needed for OIDC (npm/Python Trusted Publishing + future integrations)
      actions: write     # Required for workflow management
      contents: write    # Required for GitHub releases
      packages: write    # Required for Docker/GHCR publishing
    with:
      tool: npm
      # ... other inputs
```

**Why is `id-token: write` always required?**
- Currently used for npm and Python Trusted Publishing (no NPM_TOKEN or UV_TOKEN needed!)
- Planned for future OIDC integrations with other publishing targets (Docker registries, etc.)
- Due to GitHub Actions limitations, permissions cannot be conditionally granted in reusable workflows
- Must be set at the top-level calling workflow, even if you're not publishing to npm or PyPI

#### Required Secrets

Add these secrets to your repository settings based on your publishing targets:

```yaml
# For Docker publishing
GITHUB_TOKEN: # Auto-provided by GitHub

# For npm publishing - NO NPM_TOKEN NEEDED!
# Uses Trusted Publishing (Provenance) with OIDC
# Requires: id-token: write permission (see above)

# For Python publishing - NO UV_TOKEN NEEDED!
# Uses Trusted Publishing (Provenance) with OIDC
# Requires: id-token: write permission (see above)

# For Firefox extensions
AMO_API_KEY: # Mozilla Add-ons API key
AMO_API_SECRET: # Mozilla Add-ons API secret

# For Android builds
ANDROID_STOREPASS: # Android keystore password

# For Nx Cloud (optional)
NX_CLOUD_ACCESS_TOKEN: # Nx Cloud access token
```

### 2. Release-it Configuration for npm Publishing

When using `release-it` with Trusted Publishing, you need to configure it to skip npm's built-in checks since the workflow handles authentication via OIDC.

Create a `.release-it.json` file in your project root:

```json
{
  "npm": {
    "skipChecks": true
  }
}
```

**Why is this needed?**
- Release-it normally checks for npm authentication before publishing
- With Trusted Publishing, authentication happens automatically via GitHub's OIDC token
- `skipChecks: true` tells release-it to trust the workflow's authentication

### 3. Project Structure Examples

#### Node.js Project with npm Publishing
```
project/
├── package.json
├── .release-it.json     # Required for release-it + Trusted Publishing
├── src/
├── dist/
├── Dockerfile (optional)
└── .github/workflows/ci.yml
```

#### Node.js Project (without release-it)
```
project/
├── package.json
├── src/
├── dist/
├── Dockerfile (optional)
└── .github/workflows/ci.yml
```

#### Python Project
```
project/
├── pyproject.toml
├── uv.lock
├── src/
├── dist/
└── .github/workflows/ci.yml
```

#### Monorepo with Libraries
```
project/
├── package.json
├── packages/
│   ├── lib1/package.json
│   └── lib2/package.json
├── apps/
└── nx.json
```

## 🛡️ Security Features

### Enhanced Security
- ✅ Updated to latest action versions (checkout@v5, setup-node@v5)
- ✅ Minimal permissions (contents: read by default)
- ✅ Early secret validation with categorized exit codes

### Input Validation
- ✅ JSON validation for Docker metadata
- ✅ Library name sanitization
- ✅ Path traversal prevention

### Access Control
- ✅ Minimal required permissions
- ✅ Secret-based conditional execution
- ✅ Artifact existence validation

### Timeouts
- ✅ Optimized timeouts (5-60 minutes)
- ✅ Prevents runaway builds
- ✅ Resource usage optimization

## 🚀 Performance Optimizations

### Enhanced Performance
- ✅ Optimized timeouts for faster feedback
- ✅ Conditional Playwright setup (only when needed)
- ✅ Comprehensive workflow summary with status reporting
- ✅ Refactored summary workflow (67% code reduction)

### Caching Strategy
- ✅ Multi-language dependency caching
- ✅ Build tool caches (npm, pip, gradle, maven)
- ✅ Cross-platform cache keys
- ✅ Tool-specific cache keys for optimal isolation

### Conditional Execution
- ✅ Branch-based deployment
- ✅ Artifact-dependent publishing
- ✅ Tool-specific optimizations

### Automated Maintenance
- ✅ Dependabot configuration for weekly GitHub Actions updates
- ✅ Automated security patch application
- ✅ Reduced manual maintenance burden

## 📊 Usage Examples

### Simple Node.js App (with npm Publishing)
```yaml
name: build and publish pipeline

on:
  push:
    branches:
      - main
  pull_request:

jobs:
  build:
    name: external workflow
    uses: tehw0lf/workflows/.github/workflows/build-test-publish.yml@main
    permissions:
      id-token: write    # REQUIRED - Always needed (npm Trusted Publishing + future OIDC)
      actions: write     # Required for workflow management
      contents: write    # Required for GitHub releases
      packages: write    # Required for Docker/GHCR publishing
    with:
      tool: npm
      lint: "run lint"
      test: "run test"
      build_main: "run build"
      artifact_path: "dist"
      library_path: "dist"
      event_name: ${{ github.event_name }}
```

**Note:** This example follows the pattern from `/Coding/AI/n8n/nodes/toon` which uses Trusted Publishing for npm.

### Docker + npm Publishing
```yaml
uses: tehw0lf/workflows/.github/workflows/build-test-publish.yml@main
permissions:
  id-token: write    # REQUIRED - Always needed (npm Trusted Publishing + future OIDC)
  contents: read
  packages: write    # Required for Docker publishing to GHCR
with:
  tool: npm
  build_main: "run build"
  artifact_path: "dist"
  docker_meta: '[{"name":"my-app","file":"Dockerfile"}]'
  docker_namespace: "mycompany"
  registry: "ghcr.io"
  event_name: ${{ github.event_name }}
```

### Python Package
```yaml
uses: tehw0lf/workflows/.github/workflows/build-test-publish.yml@main
permissions:
  id-token: write    # REQUIRED - Always needed (Python Trusted Publishing)
  contents: read
with:
  tool: uv
  install: "sync"
  lint: "run lint"
  test: "run test"
  build_main: "build"
  artifact_path: "dist"
  event_name: ${{ github.event_name }}
```

### Bash Scripts
```yaml
uses: tehw0lf/workflows/.github/workflows/build-test-publish.yml@main
permissions:
  id-token: write    # REQUIRED - Always needed
  contents: read
with:
  tool: bash
  install: "install.sh"
  lint: "lint.sh"
  test: "test.sh"
  build_main: "build.sh"
  artifact_path: "dist"
  event_name: ${{ github.event_name }}
```

## 🔄 Workflow Dependencies

```mermaid
graph TD
    A[build-test-publish.yml] --> B[test-and-build.yml]
    A --> C[publish-docker-image.yml]
    A --> D[publish-npm-libraries.yml]
    A --> E[publish-python-libraries.yml]
    A --> F[publish-firefox-extension.yml]
    A --> G[release-android-apk.yml]
    A --> H[release-github.yml]
    A --> J[summarize-workflow.yml]

    C --> I[External: check-artifact + download-artifact]
    D --> I
    E --> I
    F --> I
    G --> I
    H --> I
```

## 🆘 Troubleshooting

### Common Issues

1. **Build timeouts**: Adjust timeout values in workflow files
2. **Cache misses**: Check cache key patterns and dependencies
3. **Permission errors**: Verify repository secrets and permissions
4. **Artifact not found**: Ensure `artifact_path` is correctly set

### npm Trusted Publishing Issues

#### Error: "npm ERR! need auth This command requires you to be logged in"
**Solution:** Ensure `id-token: write` permission is set in your workflow:
```yaml
permissions:
  id-token: write
```

#### Error: "release-it: npm authentication check failed"
**Solution:** Add `.release-it.json` to skip npm checks when using Trusted Publishing:
```json
{
  "npm": {
    "skipChecks": true
  }
}
```

#### Error: "Unable to get OIDC token"
**Solution:**
- Verify `id-token: write` permission is granted
- Ensure your npm package is configured for [Trusted Publishing on npmjs.com](https://docs.npmjs.com/generating-provenance-statements)
- Check that your GitHub repository has access to npm's OIDC provider

### Debug Mode

Enable debug logging by adding this secret:
```yaml
ACTIONS_STEP_DEBUG: true
```

## 📝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test with a sample project
5. Submit a pull request

## 📄 License

This workflow collection is available under the MIT License.