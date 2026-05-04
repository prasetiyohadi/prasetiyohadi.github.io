# CI/CD Lab 06: Fixing GitLab CI Cache Rules and Pre-Warming the Trivy DB

> Shipping SCA was the right call. But shipping it with two silent bugs was not. Here is how I found them and fixed them.

In the [previous post](medium/cicd-lab-refactoring-gitlab-ci-rules-and-stage-naming.md), I split the single `.on-mr-or-default` rule template into three purpose-specific ones, renamed the `sast` stage to `security`, and added auto-build on git tags. The pipeline was cleaner after that. Then I pushed a tag and noticed things running that should not have been running.

Two problems surfaced. First, the `changes:` filters on cache jobs were firing on tag pipelines even when nothing had changed. Second, the three SCA jobs were all independently downloading and caching the Trivy vulnerability database in parallel: no coordination, potential for a race condition, and redundant network time. Neither was loud. Both were wasteful.

This post covers the fixes. You can follow along at [gitlab.com/pras-labs/ci-demo](https://gitlab.com/pras-labs/ci-demo/-/tree/fix/ci-cache-rules-and-trivy-warm).

---

### Problem 1: `changes:` on Tag Pipelines

GitLab's `changes:` keyword in job `rules:` compares the current pipeline's diff against the previous state. On a merge request or push to a branch, that comparison is straightforward where the diff is the set of files changed in the event.

On a tag pipeline, there is no stable diff reference. GitLab cannot determine what changed since the last tag, so it falls back to treating every file as changed. That means a job with only a `changes:` rule and no `if:` guard will evaluate as `true` on every tag pipeline, regardless of whether any of those files actually changed.

The two cache jobs affected were `build:devenv-image` and `cache:warm`.

Before the fix, their rules looked like this:

```yaml
# build:devenv-image
rules:
  - changes:
      - .devcontainer/**/*
      - .mise.toml
  - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    changes:
      - .devcontainer/**/*
      - .mise.toml
```

```yaml
# cache:warm
rules:
  - changes:
      - package-lock.json
      - .pre-commit-config.yaml
```

The bare `changes:` block at the top of `build:devenv-image` has no `if:` condition. On a tag pipeline, it matches everything. Same for `cache:warm` where there is no guard at all, so it runs unconditionally on any event where the files appear changed.

The fix is to scope every `changes:` to a specific pipeline source:

```yaml
# build:devenv-image (after fix)
rules:
  # Rebuild only when devcontainer definition or tool versions change.
  # Scope to MR + default branch only - changes: on tag pipelines has no
  # stable diff reference and may always evaluate true, causing spurious rebuilds.
  - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    changes:
      - .devcontainer/**/*
      - .mise.toml
  - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    changes:
      - .devcontainer/**/*
      - .mise.toml
```

```yaml
# cache:warm (after fix)
rules:
  # Warm only when dependency lock files change - the cache key is file-hash-based
  # so a key change (new file hash) means old cache is unreachable by consumers anyway.
  # Scope to MR + default branch to avoid changes: ambiguity on tag pipelines.
  - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    changes:
      - package-lock.json
      - .pre-commit-config.yaml
  - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    changes:
      - package-lock.json
      - .pre-commit-config.yaml
```

Now `changes:` only runs in contexts where GitLab has a clean diff to work from. Tag pipelines skip both jobs entirely unless they are explicitly added back with their own `if:` rule. The comments in the YAML are intentional where this is the kind of subtle behavior that bites you six months later when someone adds a new rule and does not know why the `if:` is there.

---

### Problem 2: Three Jobs Downloading the Same Database

The Trivy vulnerability database is not small. When `sca:vuln`, `sca:license`, and `sca:sbom` all run in the same stage in parallel, each one independently pulls the database into cache. The original `sca:vuln` had `policy: pull-push`, meaning it would push the DB to cache after downloading it. But the other two jobs started at the same time thus there was no guarantee they would find a warm cache. In practice, all three downloaded the database from the internet on every pipeline run.

Here is what the original cache config looked like for `sca:vuln`:

```yaml
sca:vuln:
  # ...
  cache:
    - <<: *cache-trivy
      policy: pull-push
```

And the Trivy cache anchor was defined near the SCA jobs, after the SAST section, too far down to be reused by a `.pre` stage job.

---

### Fix: `cache:warm:trivy` in `.pre`

The solution is a dedicated warm job that runs in the `.pre` stage, before any other stage, and pushes the Trivy database to cache once. All SCA jobs then pull from that warm cache without pushing. No race condition. One download per pipeline.

First, I moved the `.cache-trivy` anchor to the top of the file so it is accessible from the `.pre` stage:

```yaml
# -- Reusable anchor: cache for trivy ------------------
# Reusable cache for Trivy DB. Key is static - Trivy has a built-in 24h
# freshness check and will re-download the DB when stale. The warm job in
# .pre handles the push so all parallel SCA jobs only need to pull.
# Bump the prefix (trivy-db-v2, v3…) to force a full DB reset if ever needed.
.cache-trivy: &cache-trivy
  key: trivy-db-v1
  paths:
    - .cache/trivy/
```

Then the warm job itself:

```yaml
# -- Warm Trivy vulnerability DB in .pre (push) -------------
# Runs before SAST/SCA stage so all parallel SCA jobs find a ready DB in cache.
# sca:vuln + sca:license + sca:sbom all use pull-only - no race condition.
# Trivy's built-in 24h freshness check ensures the DB is never stale.
cache:warm:trivy:
  stage: .pre
  image:
    name: aquasec/trivy:0.69.3
    entrypoint: [""]
  before_script:
    - mkdir -p .cache/trivy
  script:
    - echo "🗃️ Warming Trivy vulnerability DB cache..."
    - trivy --cache-dir .cache/trivy image --download-db-only --no-progress
  cache:
    - <<: *cache-trivy
      policy: pull-push
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

A few things worth noting here:

- **`entrypoint: [""]`:** The official Trivy Docker image sets its own entrypoint. Overriding it to empty lets GitLab run the job's `script:` directly instead of passing it as an argument to `trivy`. Without this, the job fails because GitLab tries to run the script contents as Trivy subcommands.
- **`--download-db-only`:** Tells Trivy to fetch the vulnerability database and exit. No scanning happens in this job. It runs fast and purely exists to populate the cache.
- **`--no-progress`:** Suppresses the progress bar in CI logs. Output stays clean.
- **Static cache key:** `trivy-db-v1` is intentionally static. Trivy has a built-in 24-hour freshness check: if the cached DB is more than 24 hours old, Trivy re-downloads it automatically on the next run. There is no need to rotate the cache key to keep the DB fresh. The comment explains how to force a full reset if needed, just bump the suffix.
- **Rules:** Same pattern as the fixed `build:devenv-image`: only MR and default branch. Tag pipelines do not run SCA, so there is no point warming the DB for them.

---

### Updating the SCA Jobs

With the warm job in place, all three SCA jobs need two changes: add a `needs:` on `cache:warm:trivy`, and flip the cache policy from `pull-push` to `pull`.

```yaml
sca:vuln:
  needs:
    - preflight:gate
    - cache:warm:trivy # DB is pre-warmed - pull only, no race with sca:license
  script:
    - |
      trivy fs . \
        --cache-dir .cache/trivy \
        --scanners vuln \
        --severity HIGH,CRITICAL \
        --exit-code 1 \
        --format json \
        --output $SCA_REPORT_DIR/trivy-vuln.json
    - trivy fs --cache-dir .cache/trivy --scanners vuln --severity HIGH,CRITICAL . || true
  cache:
    - <<: *cache-trivy
      policy: pull
```

```yaml
sca:license:
  needs:
    - preflight:gate
    - cache:warm:trivy # DB is pre-warmed - no race condition with sca:vuln
  script:
    - |
      trivy fs . \
        --cache-dir .cache/trivy \
        --scanners license \
        --severity HIGH,CRITICAL \
        --format json \
        # ...
  cache:
    - <<: *cache-trivy
      policy: pull
```

```yaml
sca:sbom:
  image:
    name: aquasec/trivy:0.69.3
    entrypoint: [""]
  needs:
    - preflight:gate
    - cache:warm:trivy # DB is pre-warmed
  script:
    - |
      trivy fs . \
        --cache-dir .cache/trivy \
        --format cyclonedx \
        --output $SCA_REPORT_DIR/sbom.cdx.json
  cache:
    - <<: *cache-trivy
      policy: pull
```

Two things to call out:

- **`--cache-dir .cache/trivy`:** This flag tells Trivy where to find (or write) the DB. Without it, Trivy defaults to its own internal path, which is not the directory the cache artifact is mounted at. All four jobs, the warm job and the three SCA jobs, need to use the same path.
- **`sca:sbom` image fix:** The original `sca:sbom` used `image: aquasec/trivy:0.69.3` (plain string). That worked for scanning, but it meant the entrypoint was still set to `trivy`, which breaks `script:` execution. The fix is the same `name:` + `entrypoint: [""]` pattern used in the warm job and the other SCA jobs.

The `needs:` on `cache:warm:trivy` is what enforces the ordering. GitLab will not start any SCA job until the warm job completes and the cache artifact is pushed. The SCA jobs run in parallel with each other, all reading from the same pre-populated cache location.

---

### What the Pipeline Flow Looks Like Now

```
.pre stage:
  cache:warm:trivy       <- downloads Trivy DB once, pushes to cache

preflight stage:
  preflight:secrets
  preflight:lint
  preflight:commitlint
  preflight:branch-name
  preflight:gate         <- blocks sast stage

sast/security stage (all parallel):
  sast:semgrep
  sast:bandit
  sast:secrets:trufflehog
  sast:checkov
  sca:vuln               <- pulls Trivy DB from cache (no download)
  sca:license            <- pulls Trivy DB from cache (no download)
  sca:sbom               <- pulls Trivy DB from cache (main only)
  security:gate          <- blocks build stage

build stage (manual):
  build
  build:rootless
```

`cache:warm:trivy` runs unconditionally in `.pre` on every MR and default branch pipeline. By the time the SCA jobs start, the database is already in the GitLab cache and ready to pull.

---

### Conclusion

Two quiet bugs, both fixed with targeted changes:

- **Cache rule scoping:** Added `if: $CI_PIPELINE_SOURCE == "merge_request_event"` guards to all bare `changes:` rules in `build:devenv-image` and `cache:warm`. Tag pipelines no longer trigger spurious cache rebuilds.
- **`cache:warm:trivy`:** New `.pre` stage job that downloads the Trivy vulnerability database once per pipeline and pushes it to a shared cache. All three SCA jobs flip to `policy: pull` and add `needs: [cache:warm:trivy]` to enforce ordering.
- **`--cache-dir .cache/trivy`:** Added to every `trivy fs` invocation so all jobs read from the same cache path.
- **`entrypoint: [""]`:** Added to `sca:sbom` image config to match the pattern already used in the other Trivy jobs.
- **Anchor placement:** Moved `.cache-trivy` anchor above the warm job so it is reachable from `.pre`.

The practical impact: one Trivy DB download per pipeline instead of three, no race condition between parallel SCA jobs, and no more spurious cache warming on tag pipelines.

---

### What's Next

With dependency scanning and its cache behavior cleaned up, the next step is container image scanning: running Trivy against the built Docker image to catch vulnerabilities that came in through the base image or layer installs. That closes the loop between what you declared as a dependency and what actually ended up in the artifact.

Stay tuned for next stories with Pras' Labs: The Hands-On Engineer!

---

_Tags: GitLab CI, DevSecOps, Trivy, SCA, CI/CD, Cache, Pipeline Optimization, DevOps_
