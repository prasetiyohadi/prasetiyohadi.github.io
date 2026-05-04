# CI/CD Lab: Preflight Checks and Security Pipeline in Gitlab CI

> How DevContainers, pre-commit hooks, and a GitLab CI preflight stage shift security left — and what that means for your team's velocity and risk profile.

In the [previous post](https://medium.com/@pras-labs/ci-cd-lab-building-container-image-in-gitlab-ci-d78bdcc214a9), I explored building container images in Gitlab CI using Docker-in-Docker and rootless BuildKit. That was the build stage. Now, let's go back to the beginning of the pipeline and implement the preflight checks, the first line of defense before any code gets built or deployed.

Remember the best practice CI/CD diagram from the LLM? The first phase is a set of preflight tests. In this post, I implemented the full preflight stage, a DevContainer-based development environment, and security scanning. All of this work was done across multiple commits on a feature branch and merged as a single merge request. Let me walk you through what I built and why each piece matters.

### Why Preflight Checks Matter for Business

Before diving into the implementation, let's talk about why this matters beyond just "good practice." The cost of catching a bug or security issue increases at each stage of the pipeline. A secret caught before commit costs zero. The same secret found in production means rotating every credential, auditing access logs, and potentially filing breach notifications. A Dockerfile issue caught by a linter saves hours of debugging a broken build in CI.

The idea is simple: shift everything left. Catch it as early as possible.

---

### Dev Environment as Code with DevContainers

The first thing I set up is a DevContainer. This is a Docker-based development environment that every team member can use, whether they use VS Code, JetBrains, or SSH into a cloud workspace with DevPod.

I use [mise](https://mise.jdx.dev/) as the tool version manager. Here is the `.mise.toml` that defines all the tools for this project.

```toml
[tools]
python = "3.12"
node = "22"
uv = "latest"

# Security/lint tools via aqua backend (GitHub releases)
"aqua:gitleaks/gitleaks" = "8.30.0"
"aqua:hadolint/hadolint" = "2.14.0"
"aqua:aquasecurity/trivy" = "0.69.3"

# pip tools
"pipx:pre-commit" = "latest"
```

The DevContainer Dockerfile installs mise and all tools from this manifest. The key design is that tool versions are defined in one place and shared across local development and CI.

```dockerfile
FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04

RUN apt-get update && apt-get install --no-install-recommends --yes curl git jq make \
  && rm -rf /var/lib/apt/lists/*

USER vscode
WORKDIR /home/vscode

# Install mise
RUN curl https://mise.run | sh
ENV PATH="/home/vscode/.local/bin:/home/vscode/.local/share/mise/shims:$PATH"

# Copy tool manifest first (layer cache — only rebuilds when versions change)
COPY --chown=vscode:vscode .mise.toml .

# Install all tools defined in .mise.toml
RUN mise install --yes && mise reshim
```

The `devcontainer.json` wires everything together: the Dockerfile, volume mounts for caching, and a `postCreateCommand` that runs `setup.sh` to install git hooks automatically.

```json
{
  "name": "Secure CI Dev Environment",
  "build": {
    "dockerfile": "Dockerfile",
    "context": ".."
  },
  "postCreateCommand": "git config --global --add safe.directory ${containerWorkspaceFolder} && bash .devcontainer/setup.sh",
  "postStartCommand": "pre-commit run --all-files 2>/dev/null || true"
}
```

When a developer opens the project, `setup.sh` runs automatically and installs all git hooks, npm dependencies, and pre-caches the pre-commit environments.

```bash
#!/bin/bash
set -euo pipefail

eval "$(mise activate bash)"

npm ci
pre-commit install
pre-commit install --hook-type commit-msg
pre-commit install --hook-type pre-push
pre-commit install-hooks
```

The business impact here is straightforward. New team members go from clone to productive in minutes instead of hours. There is no "Step 7: install the correct version of Node" in a setup guide. And because CI runs inside the same container image, the gap between "it works locally" and "it works in the pipeline" drops to zero.

---

### Pre-commit Hooks: Catching Issues Before They Leave the Developer's Machine

I use [pre-commit](https://pre-commit.com/) to run a suite of checks on every commit. Here is the `.pre-commit-config.yaml` configuration.

```yaml
default_stages: [pre-commit]

repos:
  # -- Secrets ---------------------------------
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks

  # -- General hygiene -------------------------
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
        args: [--allow-multiple-documents]
      - id: check-json
      - id: check-toml
      - id: check-merge-conflict
      - id: check-added-large-files
        args: [--maxkb=1000]
      - id: no-commit-to-branch
        args: [--branch, main, --branch, master]

  # -- Dockerfile ------------------------------
  - repo: https://github.com/hadolint/hadolint
    rev: v2.12.0
    hooks:
      - id: hadolint
        args: [--failure-threshold, error]

  # -- Shell scripts ---------------------------
  - repo: https://github.com/shellcheck-py/shellcheck-py
    rev: v0.10.0.1
    hooks:
      - id: shellcheck

  # -- Python ---
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.4
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

Let me explain what each group does:

- **gitleaks**: Scans every commit for API keys, tokens, passwords, and private keys. If a secret is found, the commit is rejected. Not flagged. Rejected.
- **pre-commit-hooks**: General code hygiene. Trailing whitespace, broken YAML/JSON/TOML, merge conflict markers, large files, and direct commits to main branch are all blocked.
- **hadolint**: Lints Dockerfiles for insecure or inefficient patterns.
- **shellcheck**: Catches common bash pitfalls that cause silent failures in scripts.
- **ruff**: Python linter and formatter. Adjust or remove if your project is not Python.

I also added commit message linting and branch naming convention enforcement as local hooks.

```yaml
  - repo: local
    hooks:
      - id: commitlint
        name: Lint commit message
        stages: [commit-msg]
        language: system
        entry: npx --no-install commitlint --edit
        pass_filenames: false
      - id: branch-naming
        name: Branch naming convention
        stages: [pre-push]
        language: script
        entry: .devcontainer/check-branch.sh
        pass_filenames: false
```

The `commitlint.config.js` enforces [Conventional Commits](https://www.conventionalcommits.org/) format like `feat: add user authentication` or `fix: resolve null pointer on login`. This enables automated changelogs, semantic versioning, and makes `git log` actually useful for incident investigation.

```javascript
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      'feat', 'fix', 'docs', 'style',
      'refactor', 'test', 'chore', 'ci', 'sec', 'revert'
    ]],
    'subject-max-length': [2, 'always', 100],
    'subject-case': [2, 'always', 'lower-case'],
    'body-max-line-length': [1, 'always', 200],
  }
};
```

The branch naming script enforces a `type/kebab-description` pattern (e.g., `feat/add-oidc-auth`, `fix/null-pointer-login`). It also handles Gitlab CI detached HEAD state by reading from CI environment variables instead of git.

```bash
# In GitLab CI MR pipelines, git is in detached HEAD state.
if [ -n "${CI_MERGE_REQUEST_SOURCE_BRANCH_NAME:-}" ]; then
  BRANCH="$CI_MERGE_REQUEST_SOURCE_BRANCH_NAME"
elif [ -n "${CI_COMMIT_REF_NAME:-}" ]; then
  BRANCH="$CI_COMMIT_REF_NAME"
else
  BRANCH=$(git rev-parse --abbrev-ref HEAD)
fi

PATTERN="^(feat|fix|chore|hotfix|release|sec|docs|ci)\/[a-z0-9][a-z0-9\-]*$"
```

The business impact here compounds over time. Every secret caught before push is a breach that never happens. Every formatting issue fixed automatically is a code review comment that nobody has to write. Reviewers can focus on logic and architecture instead of style nitpicks.

---

### Gitlab CI Pipeline: Enforcing the Same Checks in CI

Local hooks are great, but developers can skip them. The Gitlab CI pipeline runs the same checks as a gate that cannot be bypassed. Here is the pipeline structure in `.gitlab-ci.yml`.

```yaml
stages:
  - preflight
  - sast
  - build
  - scan
  - gate
```

First, I build and cache the DevContainer image in the Gitlab Container Registry so all CI jobs can use it.

```yaml
variables:
  DEVCONTAINER_IMAGE: $CI_REGISTRY_IMAGE/devenv:$CI_DEFAULT_BRANCH

build:devenv-image:
  stage: .pre
  image: docker:29.2.1-cli
  services:
    - docker:29.2.1-dind
  script:
    - docker pull $DEVCONTAINER_IMAGE || true
    - |
      docker build --cache-from $DEVCONTAINER_IMAGE \
        --file .devcontainer/Dockerfile \
        --tag $DEVCONTAINER_IMAGE .
    - docker push $DEVCONTAINER_IMAGE
  rules:
    - changes:
        - .devcontainer/**/*
        - .mise.toml
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

This job only runs when devcontainer files or tool versions change, or on the default branch. It uses `--cache-from` to speed up rebuilds. All other jobs use a reusable YAML anchor to run inside this image.

```yaml
.use-devenv: &use-devenv
  image: $DEVCONTAINER_IMAGE
  before_script:
    - git config --global --add safe.directory $CI_PROJECT_DIR
```

The preflight jobs run in parallel inside the devcontainer image:

```yaml
preflight:secrets:
  <<: *use-devenv
  stage: preflight
  script:
    - gitleaks detect --source . --redact --exit-code 1
  allow_failure: false

preflight:lint:
  <<: *use-devenv
  stage: preflight
  script:
    - pre-commit run --all-files --show-diff-on-failure
  allow_failure: false

preflight:commitlint:
  <<: *use-devenv
  stage: preflight
  script:
    - npm ci --prefer-offline
    - git fetch origin $CI_MERGE_REQUEST_TARGET_BRANCH_NAME 2>/dev/null || true
    - npx --no-install commitlint --from "origin/$CI_MERGE_REQUEST_TARGET_BRANCH_NAME" --to HEAD --verbose
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

preflight:branch-name:
  <<: *use-devenv
  stage: preflight
  script:
    - bash .devcontainer/check-branch.sh
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

A gate job ensures all preflight checks must pass before the pipeline continues to build and scan stages.

```yaml
preflight:gate:
  stage: preflight
  image: alpine:latest
  needs:
    - preflight:secrets
    - preflight:lint
  script:
    - echo "✅ Pre-flight gate passed"
  allow_failure: false
```

If secrets are found or linting fails, the pipeline stops. There is no "allow failure" for security.

---

### Cache Optimization: Making the Pipeline Fast

A slow pipeline is a pipeline that developers learn to bypass. I configured caching at multiple levels to keep things fast.

```yaml
.cache-npm: &cache-npm
  key:
    files:
      - package-lock.json
    prefix: npm-v1
  paths:
    - .cache/npm/
    - node_modules/

.cache-precommit: &cache-precommit
  key:
    files:
      - .pre-commit-config.yaml
    prefix: pre-commit-v1
  paths:
    - .cache/pre-commit/
```

Both caches use file-based keys, so the cache is only invalidated when the configuration actually changes. A dedicated `cache:warm` job runs in the `.pre` stage to pre-populate caches, and downstream jobs pull from cache with `policy: pull` to avoid unnecessary writes.

The Docker layer caching in the devcontainer build, the pre-commit environment caching, and the npm dependency caching all work together. After the first run, subsequent pipeline runs reuse everything that has not changed.

---

### The Makefile: Making the Right Thing Easy

I created a `Makefile` so developers do not need to remember individual commands.

```makefile
setup: ## Full local setup (run once)
	mise install
	npm ci
	$(MAKE) hooks

hooks: ## Install git hooks
	pre-commit install
	pre-commit install --hook-type commit-msg
	pre-commit install --hook-type pre-push
	pre-commit install-hooks

check: ## Run all pre-commit checks
	pre-commit run --all-files

ci-local: ## Simulate CI pre-flight locally (devcontainer CLI)
	devcontainer up --workspace-folder .
	devcontainer exec --workspace-folder . -- make check
```

Run `make setup` once, then `make check` anytime you want to validate your work locally with the same checks that CI runs. `make ci-local` even lets you simulate the full CI preflight inside the devcontainer without pushing.

---

### Conclusion

In this story, I implemented the preflight stage of the CI/CD pipeline that I started in the previous post. The key takeaway is that preflight checks should run in two places: locally via git hooks (fast feedback) and in CI (enforcement). Using a DevContainer ensures both environments are identical, so "it works on CI" equals "it works locally."

Here is a summary of the tools used:

- **gitleaks**: Secret detection in commits
- **hadolint**: Dockerfile linting
- **shellcheck**: Shell script analysis
- **ruff**: Python linting and formatting
- **commitlint**: Conventional commit message enforcement
- **pre-commit**: Hook orchestration framework
- **trivy**: Container vulnerability scanning (used in later stages)

The business impact is clear: every check that runs automatically is a class of failure that becomes impossible without human intervention. Leaked secrets are blocked, not discovered. Broken Dockerfiles are caught before build, not during incident response. Code review cycles are shorter because style and formatting are already handled.

You can check the code for this article in the repository: https://gitlab.com/pras-labs/ci-demo

### What's Next

In the next post, we will explore the SAST and container scanning stages of the pipeline using trivy and potentially trufflehog for cloud credential scanning. We could also look into running this pipeline with self-hosted Gitlab Runners to avoid quota limits on the hosted runners. Stay tuned for next stories with The Hands-On Engineer!

---

_Tags: CI/CD, DevOps, Gitlab CI, Pre-commit, Security, DevContainers, Shift Left_
