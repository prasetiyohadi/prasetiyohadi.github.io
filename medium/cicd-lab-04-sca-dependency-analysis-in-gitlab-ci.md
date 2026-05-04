# CI/CD Lab: Software Composition Analysis with Trivy in GitLab CI

> Four SAST scanners protect your code. Trivy protects everything you pulled from the internet. Here is how I added SCA, license compliance, and SBOM generation to the pipeline then cleaned up a pile of YAML debt while I was at it.

In the [previous post](https://medium.com/@pras-labs/cicd-lab-sast-static-analysis-in-gitlab-ci), I added a dedicated `sast` stage to the GitLab CI pipeline: Semgrep, Bandit, Trufflehog, and Checkov running in parallel, gated before the build stage. Those four scanners cover the code I wrote: injection patterns, hardcoded secrets, IaC misconfigurations. What they do not cover is everything I did not write: the npm packages, Python libraries, and Go modules I pulled in as dependencies. That surface area is often larger than the first-party code, and it has its own CVE history.

This post is about Software Composition Analysis (SCA), which is scanning third-party dependencies for known vulnerabilities, checking licenses, and generating a Software Bill of Materials (SBOM). I did all of it with **Trivy**, which I already had installed in the devcontainer for Dockerfile scanning. I also did a round of pipeline refactoring to reduce repetition, then added Go tooling since the project is heading that direction. Let me walk you through the changes.

You can follow along at [gitlab.com/pras-labs/ci-demo](https://gitlab.com/pras-labs/ci-demo/-/tree/feat/sca-dependency-analysis).

---

### Why SCA alongside SAST

SAST analyzes source code for patterns that indicate vulnerabilities. If I write a SQL query by string-concatenating user input, Semgrep can flag it. If I hardcode an API key in a config file, Trufflehog can find it.

SCA does something different. It looks at the dependency tree: `package-lock.json`; `go.sum`; `requirements.txt`, and cross-references the resolved packages against the NVD (National Vulnerability Database) and other advisories. If `express@4.17.1` has a path traversal CVE, that shows up in the SCA scan, not the SAST scan. The two tools cover different attack surfaces, and you need both to have meaningful coverage.

License compliance is a related concern. Pulling in a GPL-licensed library can create legal obligations you did not intend, especially in a commercial product. Trivy can flag those too.

---

### Pipeline Cleanup First

Before adding new jobs, I refactored the pipeline to reduce the repeated `rules:` blocks that had been copy-pasted into every job. Three changes made the biggest difference.

**`workflow:` rules ensure no duplicate pipelines**

By default, GitLab can trigger pipelines on both a push event and a merge request event for the same commit, running everything twice. The `workflow:` block at the top of `.gitlab-ci.yml` fixes that:

```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG
```

Only one pipeline runs per commit now. MR events, pushes to `main`, and tags each get exactly one run.

**`default:` block configures interruptible, timeout, retry**

Every job inherits these settings now:

```yaml
default:
  interruptible: true
  timeout: 15 minutes
  retry:
    max: 2
    when:
      - runner_system_failure
      - stuck_or_timeout_failure
```

- `interruptible: true` means a new push to an MR cancels the in-progress pipeline for the same MR. No more stacked pipelines burning minutes on stale commits.
- `timeout: 15 minutes` prevents a hung job from occupying a runner indefinitely.
- `retry` handles the transient runner failures that happen occasionally in shared environments.

**`.on-mr-or-default` defines one template instead of repeated rules blocks**

Every SAST and preflight job had the same two-line `rules:` block repeated verbatim. I replaced all of them with an extends template:

```yaml
.on-mr-or-default:
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

Jobs use `extends: .on-mr-or-default` instead of their own `rules:`. One change to the template updates all jobs. I also swapped `alpine:latest` for `busybox:1.37` on the gate jobs for consistent footprint and pinned version.

---

### sca:vuln Job for Dependency Vulnerability Scan

This is the core SCA job. Trivy's `fs` (filesystem) mode walks the project directory, finds lock files, and checks the resolved dependency versions against vulnerability databases.

```yaml
sca:vuln:
  stage: sast  # same stage as SAST = parallel execution
  image:
    name: aquasec/trivy:0.69.3
    entrypoint: [""]
  needs:
    - preflight:gate
  before_script:
    - mkdir -p $SCA_REPORT_DIR
  script:
    - |
      trivy fs . \
        --scanners vuln \
        --severity HIGH,CRITICAL \
        --exit-code 1 \
        --format json \
        --output $SCA_REPORT_DIR/trivy-vuln.json
    - trivy fs --scanners vuln --severity HIGH,CRITICAL . || true
  cache:
    - <<: *cache-trivy
      policy: pull-push
  extends: .on-mr-or-default
  allow_failure: false
  artifacts:
    when: always
    expire_in: 7 days
    paths:
      - reports/sca/trivy-vuln.json
    reports:
      dependency_scanning: reports/sca/trivy-vuln.json
```

A few things worth noting:

- The job runs in the `sast` stage alongside the existing SAST jobs, so everything runs in parallel. There is no extra stage just for SCA.
- `--exit-code 1` means HIGH or CRITICAL CVEs fail the job. `allow_failure: false` means that failure blocks the pipeline.
- The script runs twice: once to produce the JSON artifact, once in human-readable table format so the job log is readable in the CI UI. The second run uses `|| true` because the table output does not need to control exit code.
- `reports: dependency_scanning:` tells GitLab to parse the JSON for the Security tab in the MR view. If you have GitLab Ultimate, findings show up there directly.
- Trivy's vulnerability database is cached with `pull-push` policy. The first run downloads it; subsequent runs pull from cache. This cuts 20-30 seconds off each subsequent scan.

---

### sca:license Job for License Compliance

Dependency license issues are slower-moving than CVEs but can create real legal problems. This job identifies any dependency with a restricted or forbidden license:

```yaml
sca:license:
  stage: sast
  image:
    name: aquasec/trivy:0.69.3
    entrypoint: [""]
  needs:
    - preflight:gate
  before_script:
    - mkdir -p $SCA_REPORT_DIR
  script:
    - |
      trivy fs . \
        --scanners license \
        --severity HIGH,CRITICAL \
        --format json \
        --output $SCA_REPORT_DIR/trivy-license.json
    - |
      if grep -qE '"Category":\s*"(restricted|forbidden)"' $SCA_REPORT_DIR/trivy-license.json 2>/dev/null; then
        echo "❌ Restricted/forbidden license detected!"
        exit 1
      fi
      echo "✅ No license violations found"
  cache:
    - <<: *cache-trivy
      policy: pull
  extends: .on-mr-or-default
  allow_failure: true  # warn, don't block pipeline
  artifacts:
    when: always
    expire_in: 7 days
    paths:
      - reports/sca/trivy-license.json
```

I set `allow_failure: true` here deliberately. A restricted license finding is something I want to know about, but it usually needs a human decision, "is this actually a problem for how we're using it?" before blocking a pipeline. The artifact preserves the full report, the job goes yellow in the pipeline view, and a developer follows up. If the project matures to where license compliance is mandatory, flip `allow_failure: false`.

The cache policy is `pull` only (not `pull-push`) since `sca:vuln` already handles the DB download.

---

### sca:sbom Job for Generating Software Bill of Materials

An SBOM is a machine-readable inventory of every package in your project and its dependencies. Increasingly, this is a regulatory requirement (Executive Order 14028 in the US government context, for example) and a useful artifact for vulnerability triage when a new CVE drops. You can immediately check whether any of your projects are affected without re-scanning everything.

```yaml
sca:sbom:
  stage: sast
  image: aquasec/trivy:0.69.3
  needs:
    - preflight:gate
  before_script:
    - mkdir -p $SCA_REPORT_DIR
  script:
    - |
      trivy fs . \
        --format cyclonedx \
        --output $SCA_REPORT_DIR/sbom.cdx.json
    - echo "📦 SBOM generated (CycloneDX)"
  cache:
    - <<: *cache-trivy
      policy: pull
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  artifacts:
    expire_in: 30 days
    paths:
      - reports/sca/sbom.cdx.json
```

Two differences from the other SCA jobs:

- SBOM generation only runs on the default branch, not on every MR. An SBOM is a point-in-time snapshot of what shipped; generating one per MR commit creates noise without value.
- Artifacts expire after 30 days instead of 7. SBOMs are long-lived reference documents.

I chose CycloneDX format over SPDX because it has better Trivy support and wider tooling compatibility (Dependency-Track, grype, etc.). Both are open standards so pick whichever your toolchain prefers.

---

### security:gate Job for One Gate for All Security Jobs

Previously I had `sast:gate`. I renamed it `security:gate` and expanded its `needs:` to include the new SCA jobs:

```yaml
security:gate:
  stage: sast
  image: busybox:1.37
  needs:
    - sast:semgrep
    - sast:secrets:trufflehog
    - sast:checkov
    - sca:vuln
    - job: sca:license
      optional: true
    - job: sca:sbom
      optional: true
  script:
    - echo "✅ Security gate passed - all SAST + SCA checks complete"
  extends: .on-mr-or-default
  allow_failure: false
```

- `sca:vuln` is a hard requirement, the gate fails if vuln scan fails.
- `sca:license` and `sca:sbom` are `optional: true` because they do not always run (license allows failure, sbom runs main-only). Optional means the gate does not wait for them if they are not triggered.
- Both `build` and `build:rootless` now have `needs: [security:gate]`, so neither runs until the full security sweep is clear. I also added `resource_group:` to each build job to prevent concurrent builds on the same ref from racing.

---

### Go Tooling

The project is adding Go code, so I wired up the tooling now rather than later.

In `.mise.toml`:

```toml
[tools]
go = "1.25"
python = "3.12"
node = "22"
uv = "latest"

"aqua:golangci/golangci-lint" = "2.1.6"
"aqua:gitleaks/gitleaks" = "8.30.0"
"aqua:hadolint/hadolint" = "2.14.0"
"aqua:aquasecurity/trivy" = "0.69.3"
```

In `.pre-commit-config.yaml`, I added local hooks for Go formatting and vetting that run before every commit:

```yaml
- id: go-fmt
  name: gofmt
  stages: [pre-commit]
  language: system
  entry: gofmt -l -w
  types: [go]
- id: go-vet
  name: go vet
  stages: [pre-commit]
  language: system
  entry: go vet ./...
  pass_filenames: false
  types: [go]
```

`gofmt` rewrites files in place. If a file is reformatted, the commit is aborted and the developer sees the diff before re-staging. `go vet` catches common mistakes (mismatched format verbs, unreachable code, etc.) without being as opinionated as a full linter run.

The devcontainer got the `golang.go` VS Code extension with `source.organizeImports` on save so the editor stays in sync with the pre-commit hooks.

---

### Makefile: Local Parity

Every CI check should also run locally. The Makefile got a full Go section and SCA targets so the developer experience matches CI:

```makefile
# -- Go ----------------------------------------
GO_BIN := bin/ci-demo

build: ## Build Go binary
	go build -o $(GO_BIN) .

test: ## Run Go tests
	go test -race -coverprofile=coverage.out ./...

lint: ## Run golangci-lint
	golangci-lint run ./...

fmt: ## Format Go code
	gofmt -w .

fmt-check: ## Check Go formatting (CI mode)
	@test -z "$$(gofmt -l .)" || (echo "Files need formatting:"; gofmt -l .; exit 1)

# -- SCA ----------------------------------------
SCA_REPORT_DIR := reports/sca

sca: $(SCA_REPORT_DIR) ## Scan dependencies for vulnerabilities (Trivy)
	trivy fs --scanners vuln --severity HIGH,CRITICAL --exit-code 1 .
	trivy fs --scanners vuln --format json --output $(SCA_REPORT_DIR)/trivy-vuln.json .

license-check: $(SCA_REPORT_DIR) ## Check dependency licenses (Trivy)
	trivy fs --scanners license --severity UNKNOWN,HIGH,CRITICAL .
	trivy fs --scanners license --format json --output $(SCA_REPORT_DIR)/trivy-license.json .

sbom: $(SCA_REPORT_DIR) ## Generate SBOM (CycloneDX)
	trivy fs --format cyclonedx --output $(SCA_REPORT_DIR)/sbom.cdx.json .

# -- Aggregates --------------------------------
preflight: fmt-check vet lint check sast-secrets ## Quick preflight before push
	@echo "✅ Preflight passed"

all: preflight test sca sast ## Full CI simulation locally
	@echo "✅ All checks passed"

doctor: ## Check all required tools are installed
	@echo "Checking tools..."
	@command -v go >/dev/null && echo "✅ go $$(go version | awk '{print $$3}')" || echo "❌ go missing"
	@command -v golangci-lint >/dev/null && echo "✅ golangci-lint" || echo "❌ golangci-lint missing"
	@command -v pre-commit >/dev/null && echo "✅ pre-commit" || echo "❌ pre-commit missing"
	@command -v gitleaks >/dev/null && echo "✅ gitleaks" || echo "❌ gitleaks missing"
	@command -v trivy >/dev/null && echo "✅ trivy" || echo "❌ trivy missing"
	@command -v npx >/dev/null && echo "✅ node/npx" || echo "❌ node missing"
	@command -v mise >/dev/null && echo "✅ mise" || echo "❌ mise missing"
```

`make doctor` is the first thing I run in a new environment. It checks that every tool the pipeline depends on is actually installed, before a cryptic "command not found" error surfaces mid-scan. `make all` simulates the full pipeline locally: preflight, tests, SCA, SAST.

---

### What the Pipeline Looks Like Now

```
preflight stage:
  preflight:secrets        (gitleaks)
  preflight:lint           (pre-commit hooks)
  preflight:commitlint     (commit message format)
  preflight:branch-name    (branch naming convention)
  preflight:gate           <- blocks sast stage

sast stage (all parallel):
  sast:semgrep             (multi-language SAST)
  sast:bandit              (Python SAST)
  sast:secrets:trufflehog  (deep secret scan)
  sast:checkov             (IaC SAST)
  sca:vuln                 (dependency CVEs)    <- NEW
  sca:license              (license compliance)  <- NEW
  sca:sbom                 (SBOM, main only)     <- NEW
  security:gate            <- blocks build stage  <- RENAMED + EXPANDED

build stage (manual):
  build                    (docker build + push)
  build:rootless           (buildkit rootless)
```

The build stage only runs manually, and only after `security:gate` passes. The gate now covers both SAST findings in first-party code and known CVEs in dependencies.

---

### Conclusion

Here is what this iteration added:

- **Trivy**: `fs` mode for dependency scanning, license checking, and SBOM generation
- **`sca:vuln`**: scans `package-lock.json`, `go.sum`, and other lock files for HIGH/CRITICAL CVEs; blocks the pipeline on findings
- **`sca:license`**: flags restricted or forbidden licenses; warns without blocking (configurable)
- **`sca:sbom`**: generates a CycloneDX SBOM on the default branch; 30-day artifact retention
- **`security:gate`**: unified gate covering all SAST and SCA jobs; both build variants wait for it
- **Go tooling**: `go`, `golangci-lint` via mise; `gofmt` and `go vet` pre-commit hooks; full Makefile target set
- **Pipeline hygiene**: `workflow:` rules eliminate duplicate pipelines; `default:` block adds interruptible, timeout, retry; `.on-mr-or-default` removes repeated rules blocks

SAST and SCA together give you coverage over the full codebase: code you wrote and code you imported. The `security:gate` pattern enforces that both pass before anything ships.

---

### What's Next

With dependency scanning in place, the natural next step is container image scanning, running Trivy against the built Docker image to catch vulnerabilities that came in through the base image or layer installs. That closes the loop between dependency scanning (what you declared) and image scanning (what actually ended up in the artifact). I'll cover that in the next post.

Stay tuned for next stories with Pras' Labs: The Hands-On Engineer!

---

_Tags: GitLab CI, DevSecOps, SCA, Software Composition Analysis, Trivy, SBOM, CycloneDX, Supply Chain Security, CI/CD, Go_
