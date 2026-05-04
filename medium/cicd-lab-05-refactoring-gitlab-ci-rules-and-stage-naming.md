# CI/CD Lab 05: Cleaning Up GitLab CI Rules Where One Template Was Doing Too Much

> One rule template, applied everywhere. It worked, until it didn't. Here is how I split it into three purpose-specific templates and wired up tag-based releases in the process.

In the [previous post](https://medium.com/@pras-labs/cicd-lab-sca-dependency-analysis-in-gitlab-ci), I added Software Composition Analysis with Trivy: dependency CVE scanning, license compliance, and SBOM generation. At the end of that post, the pipeline had a `.on-mr-or-default` template applied uniformly to every job, from lint checks to security scans to artifact generation. It works. But it is semantically wrong, and that eventually causes real problems.

This post is about fixing that. I renamed the `sast` stage to `security`, split the single rule template into three purpose-specific ones, made the preflight gate work correctly outside of MR pipelines, and added auto-build on git tags. No new tools, no new scanners, just the YAML that was lying about what it was doing.

You can follow along at [gitlab.com/pras-labs/ci-demo](https://gitlab.com/pras-labs/ci-demo/-/tree/feat/sast-sca-workflow).

---

### The Problem With `.on-mr-or-default`

Here is what the original template looked like:

```yaml
.on-mr-or-default:
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

Every job: lint; commit message validation; Semgrep; Trivy; SBOM, inherited this. The logic is: run on MRs and on the default branch. That sounds reasonable until you think about what each job actually needs.

- `preflight:lint` checks code formatting. That only makes sense when someone is proposing a change. Running it on the main branch after a merge, or on a release tag, adds noise and catches nothing new.
- The `sca:sbom` job generates a Software Bill of Materials for traceability. That should happen on main pushes _and_ on release tags, but not necessarily on every MR.
- `sast:semgrep` and the other security scans should run on MRs, on main, and on tags. If you cut a release tag and the security gate does not fire, you are shipping a build that was never scanned for that specific commit.

One template cannot express all of those requirements. It was doing the right thing for some jobs by accident.

---

### Stage Rename: `sast` -> `security`

The first change is cosmetic but meaningful:

```yaml
stages:
  - preflight
  - security # SAST + SCA parallel scan
  - build
```

The previous stage was called `sast`. That made sense when it only held SAST jobs. After adding SCA (Trivy vulnerability scanning, license compliance, SBOM), the name was wrong. SCA is not SAST. They cover different surfaces, use different techniques, and produce different report types. Calling the stage `security` makes it self-documenting: everything security-related runs here, both static analysis and composition analysis.

Renaming a stage costs nothing, it is a string in the `stages:` list and a `stage:` field on each job. It does pay off every time someone reads the pipeline output and wonders why a Trivy scan appears under a stage called `sast`.

---

### Three Templates Instead of One

Here is the full replacement:

```yaml
# Only on MR (code quality checks)
.on-mr-only:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

# Security scans: MR + main + tags
.on-security-scan:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG

# Release artifacts: main + tags only
.on-release:
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG
```

The principle is simple: the trigger scope of a job should match its purpose.

**`.on-mr-only`** applies to `preflight:lint`, `preflight:commitlint`, and `preflight:branch-name`. These are all code-quality gates where they enforce formatting conventions, commit message standards, and branch naming rules. Those only matter when code is being proposed for review. After the merge, the code is in the repo and those checks are moot.

**`.on-security-scan`** applies to all SAST and SCA jobs: Semgrep, Trufflehog, Checkov, Bandit, Trivy vulnerability scan, and Trivy license check. Security gates should fire everywhere: on the MR to catch issues before merge, on main to validate the merged state, and on tags to validate what is about to become a release artifact. If a CVE surfaces in a dependency between the MR and the tag, you want to know before that tag gets built and pushed.

**`.on-release`** applies to `sca:sbom`. SBOM generation is a release artifact, not a development-cycle gate. You want it on main (continuous traceability) and on every tag (snapshot of what shipped), but not on every MR, you do not archive a bill of materials for a feature branch.

The jobs that use each template now say exactly what they mean:

```yaml
preflight:lint:
  extends: .on-mr-only # only check formatting on MR

sast:semgrep:
  extends: .on-security-scan # scan on MR, main, AND tags

sca:sbom:
  extends: .on-release # generate SBOM on main + tags
```

The inline comments are redundant with the template name, but they make the intent visible without requiring the reader to look up the template definition.

---

### Fixing the Preflight Gate for Non-MR Pipelines

The `preflight:gate` job blocks the security stage where nothing in `security` runs until the gate passes. Before this change, the gate listed `preflight:lint`, `preflight:commitlint`, and `preflight:branch-name` as hard dependencies:

```yaml
preflight:gate:
  needs:
    - preflight:secrets
    - preflight:lint
    - preflight:commitlint
    - preflight:branch-name
```

After moving those three jobs to `.on-mr-only`, they do not run on main or tag pipelines. A hard `needs:` dependency on a job that does not exist fails the pipeline immediately. The fix is marking them as optional:

```yaml
preflight:gate:
  needs:
    - preflight:secrets
    - job: preflight:lint
      optional: true
    - job: preflight:commitlint
      optional: true
    - job: preflight:branch-name
      optional: true
  extends: .on-security-scan # gate must exist for SAST to proceed
```

`preflight:secrets` (Gitleaks) is still a hard dependency because it now uses `.on-security-scan` so that secrets detection runs everywhere, always. The other three are optional, meaning the gate will proceed if those jobs were skipped rather than waiting indefinitely for jobs that will never start.

The gate itself also moves from `.on-mr-or-default` to `.on-security-scan`, so it exists on tag pipelines where the security jobs need something to depend on.

---

### Auto-Build on Git Tags

The build jobs previously had `when: manual` applied at the job level, which meant every trigger: MR; main push; tag, required a manual click to start a build. That is fine for development pipelines. For release tags, it means shipping requires a human to remember to click a button after tagging.

The updated rules replace the job-level `when: manual` with per-trigger rules:

```yaml
build:
  resource_group: build-$CI_COMMIT_REF_SLUG
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: manual
    - if: $CI_COMMIT_TAG # auto-build on tags!
      when: on_success
```

MR and main branch builds remain manual where you do not need an image built for every review cycle or every push to main. A release tag triggers an automatic build as soon as the security gate passes. The `resource_group` lock ensures only one build runs at a time for a given ref, preventing concurrent image pushes from the same source.

The same pattern applies to `build:rootless`. Both build variants behave consistently: manual everywhere except tags.

This wires up a minimal release workflow: push a tag, the security stage runs, and if everything passes the image is built and pushed automatically. No manual gate, no forgotten click.

---

### What the Pipeline Looks Like Now

```text
preflight stage:
  preflight:secrets        (gitleaks - MR, main, tags)
  preflight:lint           (pre-commit hooks - MR only)
  preflight:commitlint     (commit message format - MR only)
  preflight:branch-name    (branch naming convention - MR only)
  preflight:gate           <- blocks security stage (MR, main, tags)

security stage (all parallel):
  sast:semgrep             (multi-language SAST - MR, main, tags)
  sast:secrets:trufflehog  (deep secret scan - MR, main, tags)
  sast:checkov             (IaC SAST - MR, main, tags)
  sast:bandit              (Python SAST - MR, main, tags)
  sca:vuln                 (dependency CVEs - MR, main, tags)
  sca:license              (license compliance - MR, main, tags)
  sca:sbom                 (SBOM - main, tags only)
  security:gate            <- blocks build stage (MR, main, tags)

build stage:
  build                    (docker build + push - manual on MR/main, auto on tags)
  build:rootless           (buildkit rootless - manual on MR/main, auto on tags)
```

---

### Conclusion

Here is what changed in this iteration:

- **Stage rename**: `sast` -> `security`; accurate now that SAST and SCA both live there
- **`.on-mr-only`**: lint, commitlint, branch-name now only run on MR pipelines
- **`.on-security-scan`**: all security jobs run on MR, main, and tags; closes the gap on tag-based releases
- **`.on-release`**: SBOM generation runs on main and tags; not on every MR
- **Optional gate dependencies**: `preflight:gate` no longer fails on non-MR pipelines when quality-only jobs are skipped
- **Tag auto-build**: both build variants fire automatically on `$CI_COMMIT_TAG`; manual trigger preserved for MR and main

The pipeline now says what it means. Each job's trigger scope matches its purpose, and cutting a release tag gives you a full security scan plus automatic image build with no manual intervention.

---

### What's Next

With the pipeline semantics cleaned up, the next step is container image scanning which is running Trivy against the built Docker image to catch vulnerabilities that came in through the base image or layer installs. That closes the loop between dependency scanning (what you declared in lock files) and image scanning (what actually ended up in the built artifact). I'll cover that in the next post.

Stay tuned for next stories with Pras' Labs: The Hands-On Engineer!

---

_Tags: GitLab CI, DevSecOps, CI/CD, Pipeline Refactoring, YAML, Security Scanning, SAST, SCA, GitLab Rules, Release Automation_
