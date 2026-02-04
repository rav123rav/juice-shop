# Security Audit Agent — GitHub-native Security Features

This agent file documents recommended rules and concrete steps to enable and maintain GitHub-native security features for the repository. Use these instructions when reviewing commits/changes or preparing repository configuration so that native GitHub security tooling is enabled, consistent, and effective.

IMPORTANT: Some repository-level features (GitHub Advanced Security, Secret Scanning push protection, etc.) require organization-level settings, billing, or repository admin permissions.

## Overview — Recommended GitHub security features

Enable and configure the following GitHub features (where available) to improve security posture and automate security maintenance:

- Dependabot (version updates + security updates)
- Dependabot alerts and the dependency graph
- Secret scanning and Secret scanning push protection
- Code scanning (CodeQL) — CodeQL analysis workflow for code scanning
- SBOM generation (Software Bill of Materials) via CI
- GitHub Advanced Security (enable Code scanning, secret scanning, and other capabilities)
- Branch protection rules (require status checks and code scanning)
- Dependabot PR automerge for low-risk security updates (optional policy)
- Automated workflows to surface and triage findings (issue templates, labels)

## Quick admin checklist for repository owners / admins

- [ ] Turn on "Dependabot alerts" and "Dependency graph" (Settings > Security & analysis).
- [ ] Turn on "Dependabot security updates" if you want automatic security updates (Settings > Security & analysis).
- [ ] Turn on "Secret scanning" (Settings > Security & analysis).
- [ ] Enable "Secret scanning push protection" if available for the repo/org (prevents commits that leak secrets).
- [ ] Enable GitHub Advanced Security for the repository/org (required to fully use some features).
- [ ] Add a CodeQL code-scanning workflow to `.github/workflows/`.
- [ ] Add a reproducible SBOM generation workflow to `.github/workflows/`.
- [ ] Add a Dependabot config to `.github/dependabot.yml`.
- [ ] Add branch protection rules requiring code scanning & CI checks before merge.

## How to enable these features (admin steps)

1. Open the repository on GitHub (must be an admin).
2. Go to Settings → Security & analysis.
3. Enable:
   - Dependency graph
   - Dependabot alerts
   - Dependabot security updates (if desired)
   - Secret scanning
   - Push protection for secrets (if available and desired)
4. If your organization has a GitHub Advanced Security license, enable GitHub Advanced Security for this repository (Settings → Code security and analysis → Enable GitHub Advanced Security). This enables additional controls and integrations.
5. Add the recommended configuration files below to this repository to enable automation.

## Recommended configuration files (examples)

Add the following example files to the repo to activate automation and show reviewers what to expect. Adapt paths and package ecosystems as needed.

Example Dependabot config: `.github/dependabot.yml`

```yaml
version: 2
updates:
  # Backend / root package.json (Node)
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5
    # Create PRs for all version updates. To only open security fixes, use vulnerabilities-only: true
    vulnerabilities-only: false
    # Optionally set target-branch if using development branch (e.g. develop)
    # target-branch: "develop"

  # Frontend package.json (frontend folder)
  - package-ecosystem: "npm"
    directory: "/frontend"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5
    vulnerabilities-only: false
```

Notes:
- To limit Dependabot to only security fixes, set `vulnerabilities-only: true`.
- Adjust `directory` and `package-ecosystem` entries for other package types (pip, composer, maven, etc).

Example CodeQL (Code scanning) workflow: `.github/workflows/codeql-analysis.yml`

```yaml
name: "CodeQL"
on:
  push:
    branches: [ "develop", "main" ]
  pull_request:
    # The branches to run on for PRs
    branches: [ "develop", "main" ]
  schedule:
    - cron: '0 3 * * 0' # weekly scan

jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: javascript
          # If you want to use a custom config file:
          # config-file: .github/codeql/codeql-config.yml

      - name: Build (install deps)
        run: |
          npm ci
          npm run build --if-present

      - name: Run CodeQL analysis
        uses: github/codeql-action/analyze@v2
```

Example SBOM generation workflow using CycloneDX action: `.github/workflows/generate-sbom.yml`

```yaml
name: "Generate SBOM"
on:
  push:
    branches: [ "develop", "main" ]
  workflow_dispatch:

jobs:
  sbom:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install dependencies
        run: |
          npm ci

      - name: Generate CycloneDX SBOM (root)
        uses: CycloneDX/cyclonedx-action@v1
        with:
          project-type: 'node'
          output-file: 'sbom/bom-root.xml'
          include-dev-dependencies: 'true'

      - name: Generate CycloneDX SBOM (frontend)
        if: exists('frontend/package.json')
        uses: CycloneDX/cyclonedx-action@v1
        with:
          project-type: 'node'
          working-directory: 'frontend'
          output-file: 'sbom/bom-frontend.xml'
          include-dev-dependencies: 'true'

      - name: Upload SBOM artifacts
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom/
```

Notes:
- The example uses CycloneDX output. You may use other SBOM generators if preferred (e.g., Syft/Anchore).
- Consider storing SBOM files as artifacts and/or publishing to a central repo or dependency governance tool.

## Secret scanning and push protection

- Enable Secret scanning in Settings → Security & analysis.
- Enable "Secret scanning push protection" (if your org has it) to stop accidental pushes of high-confidence secrets.
- Consider also adding pre-commit hooks locally to prevent accidental commits (e.g., commitlint, pre-commit with detect-secrets) — but these are complementary to GitHub's server-side scanning.

## Branch protection & required checks

Create branch protection rules (Settings → Branches) for protected branches (e.g., `develop`, `main`):

- Require status checks to pass before merging
  - Include CodeQL / code-scanning checks
  - Include CI build/test checks (lint, unit/integration tests)
- Require PR reviews before merging
- Optionally require signed commits

This ensures code scanning and SBOM/CI checks run before merges.

## Handling findings & triage workflow

- Configure notifications and CODEOWNERS to route security PRs/alerts to maintainers.
- Create issue templates or an internal security triage workflow that:
  - Labels findings (e.g., "security: high", "security: dependency").
  - Assigns triage owners.
  - Tracks remediation deadlines.
- Use Dependabot's auto-merge selectively for low-risk security updates (configure via repository settings and repo policy). Be conservative with auto-merge.

## Automation rules for PRs and issues

- Add an automation to label Dependabot PRs (e.g., `dependencies`) and a GitHub Actions job to run extra security tests on these PRs.
- Use a workflow to convert high-severity CodeQL alerts to issues or to notify a Slack channel (via webhook) for rapid response.

## Notes about GitHub Advanced Security

- GitHub Advanced Security is an organization-level feature that enables:
  - Private CodeQL analysis data retention and scheduling
  - Secret scanning for private repositories
  - Additional policies and insights
- If your organization subscribes to GitHub Advanced Security, ask org admins to enable it and verify that repository settings list it as enabled.

## Review checklist for PRs touching security configuration

When reviewing pull requests that add/change GitHub security configs, check:

- [ ] The repo has a valid `.github/dependabot.yml` that covers all relevant ecosystems.
- [ ] A CodeQL workflow is present and configured for main languages used.
- [ ] SBOM generation exists and builds successfully in CI.
- [ ] No secrets or credentials were added to the PR (use the secret-scanning results).
- [ ] Branch protection requires the new code scanning and CI checks.
- [ ] If enabling auto-merge or auto-fix automation, ensure maintainers have final approval path.
- [ ] Any added third-party GitHub Actions are vetted (use actions from trusted publishers).

## References and docs

- Dependabot configuration: https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/configuration-options-for-dependabot-version-updates
- Code scanning (CodeQL): https://docs.github.com/en/code-security/secure-your-code
- Secret scanning: https://docs.github.com/en/code-security/secret-scanning
- SBOM generation (CycloneDX action example): https://github.com/CycloneDX/cyclonedx-github-action
- GitHub Advanced Security: https://docs.github.com/en/code-security
rest
