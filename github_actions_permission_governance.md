# GitHub Actions Permission Governance Architecture

This document outlines a production-oriented strategy for automating and governing GitHub workflow permissions across repositories and organizations. The design emphasizes least-privilege, token hardening, and CI trust boundaries in a multi-repo/org-scale environment.

## 1. Strategic Model

To establish a robust permission architecture, it is crucial to clarify the control mechanisms at various layers:

| Layer          | Purpose                       | Control Mechanism              |
| :------------- | :---------------------------- | :----------------------------- |
| Organization   | Default token permission      | Org-level Actions settings     |
| Repository     | Workflow execution scope      | Repo settings                  |
| Workflow file  | Token privilege envelope      | `permissions:` block           |
| Job level      | Principle of least privilege  | `jobs.<job>.permissions`       |
| Step level     | External token isolation      | PAT / OIDC / App token         |

The overarching goal is to default to `permissions: read-all` and explicitly escalate privileges only when absolutely necessary.

## 2. Hardened Baseline Workflow Template

The `ci-secure-workflow-template.yml` provided enforces explicit permission declaration, minimal default token access, job-scoped privilege escalation, and OIDC-ready federation. This template serves as a secure starting point for individual repositories.

```yaml
name: CI Secure Workflow Template

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

# Global default: read-only
permissions: read-all

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:

  build:
    name: Build & Test
    runs-on: ubuntu-latest

    # Explicitly declare required permissions
    permissions:
      contents: read
      pull-requests: read
      checks: write   # required if updating status checks

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

  release:
    name: Release
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest

    # Elevated permissions only for this job
    permissions:
      contents: write
      packages: write
      id-token: write  # required for OIDC federation

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Create release
        uses: softprops/action-gh-release@v2
        with:
          tag_name: v${{ github.run_number }}
```

## 3. Organization-Level Enforcement Pattern

To prevent privilege drift across the organization, it is recommended to enforce a read-only default token at the organization level:

1. Navigate to: `Organization Settings → Actions → Workflow permissions`
2. Set: `Default: Read repository contents permission`
3. Disable: `Allow GitHub Actions to create and approve pull requests` (unless explicitly required).

This configuration mandates that engineers explicitly escalate permissions within their workflows, adhering to the principle of least privilege.

## 4. Centralized Reusable Workflow (Enterprise Pattern)

For organizations with numerous repositories, standardizing permission policies through reusable workflows is highly effective. The `ci-template.yml` demonstrates this approach:

**`.github/workflows/ci-template.yml`**

```yaml
name: Org CI Template

on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string

permissions: read-all

jobs:
  standardized-build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      checks: write

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}

      - run: npm ci
      - run: npm test
```

Repositories can then call this template, centralizing permission logic:

**Example Repo-level Workflow:**

```yaml
name: Repo CI

on:
  push:
    branches: [ main ]

jobs:
  call-template:
    uses: org-name/repo-name/.github/workflows/ci-template.yml@main
    with:
      node-version: "20"
```

## 5. GitHub App Token Instead of GITHUB_TOKEN (Advanced)

For enhanced control and isolation, consider using a GitHub App token instead of the default `GITHUB_TOKEN`:

1.  **Create a GitHub App**: Configure the app with precise, restricted permissions (e.g., `issues: write`, `contents: read`).
2.  **Generate Installation Token Dynamically**: Use an action like `tibdex/github-app-token@v2` to generate a short-lived token within the workflow.

```yaml
- name: Generate GitHub App Token
  id: app-token
  uses: tibdex/github-app-token@v2
  with:
    app_id: ${{ secrets.APP_ID }}
    private_key: ${{ secrets.APP_PRIVATE_KEY }}

- name: Use app token
  run: |
    curl -H "Authorization: Bearer ${{ steps.app-token.outputs.token }}" \
         https://api.github.com/repos/${{ github.repository }}/issues
```

This approach isolates automation identity from human accounts, reducing the blast radius in case of compromise.

## 6. OIDC Federation Instead of Stored Cloud Secrets

Modern best practices advocate for OIDC federation to manage cloud credentials, eliminating the need to store static secrets in GitHub:

```yaml
permissions:
  id-token: write
  contents: read
```

This allows for federation with cloud providers like AWS, Azure, or GCP. For example, with AWS:

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GitHubOIDCRole
    aws-region: eu-west-1
```

## 7. Permission Matrix Strategy (For Audit)

Documenting a permission matrix helps in auditing and understanding typical permission requirements:

| Action Type         | Required Permission |
| :------------------ | :------------------ |
| Read repo           | `contents: read`    |
| Push commit         | `contents: write`   |
| Create release      | `contents: write`   |
| Update PR           | `pull-requests: write` |
| Upload artifact     | `actions: write`    |
| OIDC                | `id-token: write`   |

This matrix should be included in the organization's engineering handbook.

## 8. Automated Enforcement via Policy Bot

To programmatically enforce explicit permission declarations, a policy bot or custom checks can be implemented. The `enforce-permissions.sh` script can be used as a pre-commit hook or within a separate workflow to check for missing `permissions:` blocks:

```bash
#!/bin/bash

# This script checks if all GitHub Actions workflow files have an explicit 'permissions:' block.
# It helps enforce least privilege by preventing implicit write-all token usage.

# Find all workflow YAML files and check for the 'permissions:' keyword.
# If any file does NOT contain 'permissions:', its name will be printed by grep -L.
# If grep -L finds any such files, it will return a non-zero exit code, causing the script to exit with an error.

UNSECURED_WORKFLOWS=$(grep -L "permissions:" .github/workflows/*.yml)

if [ -n "$UNSECURED_WORKFLOWS" ]; then
  echo "The following workflow files are missing an explicit 'permissions:' block:"
  echo "$UNSECURED_WORKFLOWS"
  echo "Please add a 'permissions:' block to these files to enforce least privilege."
  exit 1
else
  echo "All workflow files contain an explicit 'permissions:' block. Good job!"
  exit 0
fi
```

This prevents implicit `write-all` token usage and ensures adherence to security policies.

## 9. Threat Model Considerations

Understanding primary risk vectors and their mitigations is crucial for a secure GitHub Actions environment:

**Primary Risk Vectors:**

*   Pull request from fork executing write token
*   Dependency compromise in third-party action
*   Token exfiltration via malicious PR
*   Privilege escalation across reusable workflows

**Mitigations:**

*   Use `pull_request` instead of `pull_request_target` unless absolutely necessary.
*   Pin actions by SHA, not by tag, to prevent unexpected changes.
*   Enforce `read-all` as the default permission.
*   Require approval for workflows triggered by forked pull requests.
*   Restrict access to reusable workflows.

## 10. Minimal Secure Template (For Fast Adoption)

For quick adoption of a strict baseline, the following minimal template can be used:

```yaml
permissions: read-all

jobs:
  ci:
    runs-on: ubuntu-latest
    permissions:
      contents: read
```

This template defaults to `read-all` and requires intentional escalation of privileges.

## 11. Architectural Philosophy

The design of workflow permissions should mirror distributed systems trust principles:

*   Treat CI as an untrusted execution environment.
*   Scope identity per job.
*   Separate human and automation identity.
*   Federate instead of storing secrets.
*   Default deny, explicit allow.

**Least privilege is not a feature; it is a governance mechanism.**
