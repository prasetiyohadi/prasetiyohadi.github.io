# CI/CD Lab: SAST and Static Analysis in Gitlab CI

> Four scanners, a diff-aware MR pipeline, and a gate that blocks builds until security checks pass. This is what SAST looks like when you stop treating it as a checkbox.

In the [previous post](https://medium.com/@pras-labs/cicd-lab-preflight-checks-and-security-pipeline-in-gitlab-ci), I wired up the preflight stage of the pipeline: a DevContainer-based dev environment, pre-commit hooks, `gitleaks` for secrets, `hadolint` for Dockerfiles, commit message linting, and a preflight gate. That covered hygiene and surface-level scans. What it did not catch is implementation-level vulnerabilities: SQL injection patterns, insecure deserialization, hardcoded cryptographic material buried in a config, or an IaC misconfiguration that exposes a cluster.

That is what this post is about. I added a dedicated `sast` stage to the pipeline with four scanners (Semgrep, Bandit, Trufflehog, Checkov), a SAST gate, matching `make` targets so everything also runs locally, and tightened the preflight gate while I was at it. I also restructured the pipeline so `build` only runs after security passes. Let me walk you through the changes.

---

### Pipeline Restructure: Build Last, Gate Early

The first thing I changed is the stage order in `.gitlab-ci.yml`.

```yaml
stages:
  - preflight
  - sast
  - scan
  - gate
  - build
```

Previously, `build` sat in the middle of the pipeline. That meant every push ate Docker-in-Docker build minutes even if the pipeline was going to fail on a SAST finding three stages later. Now the order is: hygiene first, static analysis next, container/dependency scans after that, a final gate, and only then does the build run. The `build` job also became manual:

```yaml
build:
  stage: build
  # ... existing docker build+push
  when: manual
```

Manual gating on `build` is a deliberate choice. On a feature branch, I rarely need a pushed image from every commit. If I want one, one click is cheaper than burning compute on every WIP push. On the default branch, the same job will still run manually when I am ready to cut a tagged image.

I also added a report directory variable and a matching `.gitignore` entry so SAST artifacts land in one place without polluting commits:

```yaml
variables:
  # ...
  SAST_REPORT_DIR: $CI_PROJECT_DIR/reports/sast
```

```gitignore
# SAST reports
reports/
```

The `.use-devenv` anchor creates the directory up front so every SAST job has a known output path:

```yaml
.use-devenv: &use-devenv
  image: $DEVCONTAINER_IMAGE
  before_script:
    - git config --global --add safe.directory $CI_PROJECT_DIR
    - mkdir -p $SAST_REPORT_DIR
```

---

### Stricter Preflight Gate

Before adding SAST, I made the preflight gate strict. In the previous post, `preflight:commitlint` and `preflight:branch-name` were `allow_failure: true` and not wired into the gate. That was a compromise while I was iterating on the conventions. Both are now mandatory:

```yaml
preflight:gate:
  stage: preflight
  needs:
    - preflight:secrets
    - preflight:lint
    - preflight:commitlint
    - preflight:branch-name
  script:
    - echo "✅ Pre-flight gate passed"
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  allow_failure: false
```

I also added a `pre-push` hook that re-runs `gitleaks` right before anything leaves the developer's machine. Pre-commit catches most secrets, but pre-push is the last chance before a credential ends up in a remote:

```yaml
- id: sast-secrets-prepush
  name: Secret scan (gitleaks)
  stages: [pre-push]
  language: system
  entry: gitleaks detect --source . --redact --exit-code 1
  pass_filenames: false
```

Pre-push is cheap (seconds) and the cost of a leaked key reaching GitLab origin is not.

---

### Semgrep: Multi-Language SAST

Semgrep is the workhorse of this stage. It runs OWASP Top 10 rules plus curated packs for secrets, CI configs, Dockerfiles, and Kubernetes manifests. One tool, five rule packs, multiple languages.

```yaml
sast:semgrep:
  <<: *use-devenv
  stage: sast
  image: semgrep/semgrep:latest
  needs:
    - preflight:gate
  script:
    - |
      # On MR: scan only changed files for speed
      if [ -n "$CI_MERGE_REQUEST_IID" ]; then
        git fetch origin $CI_MERGE_REQUEST_TARGET_BRANCH_NAME
        CHANGED=$(git diff --name-only "origin/$CI_MERGE_REQUEST_TARGET_BRANCH_NAME"..HEAD)
        echo "🔎 Scanning changed files only:"
        echo "$CHANGED"
        semgrep scan \
          --config "p/owasp-top-ten" \
          --config "p/secrets" \
          --config "p/ci" \
          --config "p/dockerfile" \
          --config "p/kubernetes" \
          --json-output $SAST_REPORT_DIR/semgrep.json \
          --sarif-output $SAST_REPORT_DIR/semgrep.sarif \
          --error \
          $CHANGED
      else
        semgrep scan \
          --config "p/owasp-top-ten" \
          --config "p/secrets" \
          --config "p/ci" \
          --config "p/dockerfile" \
          --config "p/kubernetes" \
          --json-output $SAST_REPORT_DIR/semgrep.json \
          --sarif-output $SAST_REPORT_DIR/semgrep.sarif \
          --error \
          .
      fi
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  allow_failure: false
  artifacts:
    when: always
    expire_in: 7 days
    paths:
      - reports/sast/semgrep.sarif
    reports:
      sast: reports/sast/semgrep.sarif # shows in GitLab Security tab
```

A few things to note:

- **Diff-aware on MRs**: On merge requests, I fetch the target branch and scan only changed files. A full repo scan can take minutes on a large codebase; a PR scan takes seconds. On the default branch, I scan everything as a safety net.
- **Rule packs**: `p/owasp-top-ten` is the obvious one, but `p/ci`, `p/dockerfile`, and `p/kubernetes` pay for themselves the first time someone commits a `privileged: true` Pod spec or a Dockerfile that runs as root.
- **SARIF artifact**: The `reports.sast` key tells GitLab to ingest the SARIF file and render findings in the Security tab. No extra integration needed.
- **`--error` exits non-zero on findings**: `allow_failure: false` means a finding stops the pipeline.

---

### Trufflehog: Verified-Only Deep Secret Scan

`gitleaks` is fast and regex-based. It runs on every commit. `trufflehog` is slower but deeper: it walks git history and, crucially, **verifies** credentials by attempting a live check against the provider (AWS, GitHub, Slack, etc). `--only-verified` means we only fail the pipeline on secrets that actually work.

```yaml
sast:secrets:trufflehog:
  <<: *use-devenv
  stage: sast
  image:
    name: trufflesecurity/trufflehog:latest
    entrypoint: [""]
  needs:
    - preflight:gate
  script:
    - |
      if [ -n "$CI_MERGE_REQUEST_IID" ]; then
        git fetch origin $CI_MERGE_REQUEST_TARGET_BRANCH_NAME 2>/dev/null || true
        git branch $CI_MERGE_REQUEST_TARGET_BRANCH_NAME origin/$CI_MERGE_REQUEST_TARGET_BRANCH_NAME
        trufflehog git file://. \
          --since-commit "origin/$CI_MERGE_REQUEST_TARGET_BRANCH_NAME" \
          --only-verified \
          --json \
          --fail \
          --no-update \
          > $SAST_REPORT_DIR/trufflehog.json || EXIT=$?
      else
        trufflehog git file://. \
          --only-verified \
          --json \
          --fail \
          --no-update \
          > $SAST_REPORT_DIR/trufflehog.json || EXIT=$?
      fi
```

Using two secret scanners is not redundant. They fail differently: `gitleaks` is great at catching high-entropy strings and known formats fast; `trufflehog` validates. A candidate blocked by `gitleaks` is a guess. A candidate blocked by `trufflehog --only-verified` is a working key that is already burning.

The `--since-commit` flag on MRs scans only commits introduced by the branch, which keeps runtime reasonable on long-lived repos.

---

### Bandit: Python SAST, Conditional

Bandit is Python-specific. I did not want the job to fail loudly on a repo with zero Python files, so the script guards itself:

```yaml
sast:bandit:
  <<: *use-devenv
  stage: sast
  needs:
    - preflight:gate
  script:
    - |
      # Only run if Python files exist
      PY_FILES=$(find . -name "*.py" \
        -not -path "./.venv/*" \
        -not -path "./node_modules/*" \
        -not -path "./.git/*" | head -5)

      if [ -z "$PY_FILES" ]; then
        echo "⚠️ No Python files found, skipping bandit"
        exit 0
      fi

      bandit \
        --recursive \
        --format json \
        --output $SAST_REPORT_DIR/bandit.json \
        --severity-level medium \
        --exclude .venv,node_modules,.git \
        . || EXIT=$?
```

- **`--severity-level medium`**: Bandit's low-severity bucket is heavy on style flags that duplicate what `ruff` already catches. Medium and above is where the real findings live.
- **Skip path**: `.venv`, `node_modules`, and `.git` are excluded so Bandit does not waste time scanning third-party code or lint Python inside VCS metadata.
- **`optional: true` on the gate**: Since the job can no-op, the SAST gate treats it as optional. The other three scanners are always required.

---

### Checkov: IaC Scanning

Checkov covers what code-focused scanners miss: infrastructure-as-code misconfigurations. Dockerfile, Kubernetes manifest, and GitLab CI config.

```yaml
sast:checkov:
  <<: *use-devenv
  stage: sast
  needs:
    - preflight:gate
  script:
    - |
      checkov \
        --directory . \
        --output cli \
        --output sarif \
        --output-file-path $SAST_REPORT_DIR \
        --skip-check CKV_DOCKER_2,CKV_DOCKER_7 \
        --framework dockerfile,kubernetes,gitlab_ci \
        --compact \
        --soft-fail-on MEDIUM \
        --hard-fail-on HIGH,CRITICAL
  artifacts:
    when: always
    expire_in: 7 days
    paths:
      - reports/sast/*
    reports:
      sast: reports/sast/results_sarif.sarif
```

Two things worth calling out:

- **`--soft-fail-on MEDIUM --hard-fail-on HIGH,CRITICAL`**: Medium findings show up in the Security tab but do not block the pipeline. High and Critical do. This is a pragmatic split: a medium is worth a reviewer seeing, a critical should never merge.
- **`--skip-check CKV_DOCKER_2,CKV_DOCKER_7`**: `CKV_DOCKER_2` (healthcheck) and `CKV_DOCKER_7` (pinned versions on apt-get) are noisy in a devcontainer context. If you are using Checkov on production images, you probably want those back on.

Like Semgrep, Checkov's SARIF output is wired into GitLab's Security tab via the `reports.sast` key.

---

### The SAST Gate

Each scanner fails on its own if it finds something bad. The gate's job is to make the stage atomic: nothing downstream runs until every required scanner passes.

```yaml
sast:gate:
  stage: sast
  image: alpine:latest
  needs:
    - sast:semgrep
    - sast:secrets:trufflehog
    - sast:checkov
    # bandit optional if no Python:
    - job: sast:bandit
      optional: true
  script:
    - echo "✅ SAST gate passed - unblocking scan stage"
  <<: *sast-rules
  allow_failure: false
```

The `.sast-rules` anchor keeps the rule list DRY across jobs:

```yaml
.sast-rules: &sast-rules
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

With the gate in place, any downstream job (container scan, build) can simply `needs: [sast:gate]` to inherit the full SAST contract without listing individual scanners.

---

### Running SAST Locally: Makefile Targets

The pipeline is pointless if developers cannot reproduce it locally. I added matching `make` targets so anyone can run any scanner (or all of them) without pushing.

```makefile
sast-secrets: $(SAST_REPORT_DIR) ## Scan for secrets (gitleaks + trufflehog)
	@echo "-> gitleaks..."
	gitleaks detect --source . --redact --verbose
	@echo "-> trufflehog..."
	trufflehog git file://. --only-verified --no-update

sast-semgrep: $(SAST_REPORT_DIR) ## Run Semgrep SAST
	@echo "-> semgrep..."
	docker run --rm -v "$(PWD):/src" semgrep/semgrep:latest \
	 semgrep scan \
	  --config "p/owasp-top-ten" \
	  --config "p/secrets" \
	  --config "p/dockerfile" \
	  --config "p/kubernetes" \
	  --json-output /src/$(SAST_REPORT_DIR)/semgrep.json \
	  --error \
	  /src

sast-diff: $(SAST_REPORT_DIR) ## SAST on changed files only (mirrors MR pipeline)
	@echo "-> Scanning changed files vs main..."
	@git fetch origin main 2>/dev/null; \
	CHANGED=$$(git diff --name-only origin/main..HEAD 2>/dev/null || \
	 git diff --name-only HEAD~1..HEAD); \
	echo "Changed files: $$CHANGED"; \
	docker run --rm -v "$(PWD):/src" semgrep/semgrep:latest \
	 semgrep scan \
	  --config "p/owasp-top-ten" \
	  --config "p/secrets" \
	  --error \
	  $$CHANGED

sast: sast-secrets sast-semgrep sast-bandit sast-checkov ## Run all SAST checks
	@echo "✅ All SAST checks done - reports in $(SAST_REPORT_DIR)/"

sast-report: ## Summarize last SAST run
	@echo "-- SAST Report Summary -----------------"
	@[ -f $(SAST_REPORT_DIR)/semgrep.json ] && \
		echo "Semgrep: $$(cat $(SAST_REPORT_DIR)/semgrep.json | \
		python3 -c 'import sys,json; \
		d=json.load(sys.stdin); \
		print(str(len(d.get("results",[])))+" findings")')" \
		|| echo "Semgrep: no report yet"
```

The one I reach for most is `make sast-diff`. It mirrors exactly what the MR pipeline does, so I can see if a change will land cleanly before pushing. `make sast-report` is a dumb-but-useful summarizer over the JSON output from the last run.

---

### Tooling Additions

`.mise.toml` picks up three new tools so local and CI use the same versions:

```toml
"aqua:trufflesecurity/trufflehog" = "3.93.8"

# pip tools
"pipx:pre-commit" = "latest"
"pipx:bandit" = "latest"
"pipx:checkov" = "latest"
```

Semgrep runs via its official container image in CI and in the `make` target, which keeps it pinned to the image tag rather than the `mise` lockfile. I also added the official VSCode extension to the devcontainer for inline Semgrep feedback while editing:

```json
"extensions": [
  "ms-azuretools.vscode-docker",
  "gitlab.gitlab-workflow",
  "streetsidesoftware.code-spell-checker",
  "eamodio.gitlens",
  "semgrep.semgrep"
]
```

---

### Conclusion

The SAST stage is the first layer of the pipeline that actually reads the code for what it does, not just what it looks like. Preflight catches the obvious (secrets, lint, Dockerfile smells). SAST catches what only a rule engine that understands code semantics can: injection patterns, insecure serialization, risky IaC defaults, and working credentials in git history.

Here is the tool recap:

- **Semgrep**: Multi-language SAST with OWASP, secrets, CI, Dockerfile, and Kubernetes rule packs. SARIF feeds GitLab's Security tab.
- **Bandit**: Python-specific SAST, conditionally run when `.py` files exist.
- **Trufflehog**: Verified-only deep secret scan over git history. Complements `gitleaks`.
- **Checkov**: IaC scanning across Dockerfile, Kubernetes, and GitLab CI with a soft/hard severity split.
- **SAST gate**: Atomic stage gate so the `build` stage only runs after all scanners pass.

The practical impact: the `build` stage does not spend a single Docker-in-Docker minute until the entire preflight and SAST chain has passed. Security becomes a precondition of compute, not a parallel task that sometimes gets ignored.

You can check the code for this article here: [https://gitlab.com/pras-labs/ci-demo/-/tree/feat/sast-static-analysis](https://gitlab.com/pras-labs/ci-demo/-/tree/feat/sast-static-analysis)

### What's Next

In the next post, I will cover the `scan` stage: container image scanning with `trivy` for CVEs and SBOM generation, and configuration scans on the built image itself. After that, I want to look at self-hosted GitLab Runners so we are not rate-limited by hosted runner minutes on bigger pipelines. Stay tuned for next stories with The Hands-On Engineer!

---

_Tags: CI/CD, DevOps, Gitlab CI, SAST, Security, Shift Left, Semgrep_
