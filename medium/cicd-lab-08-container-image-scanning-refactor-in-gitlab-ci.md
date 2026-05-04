# CI/CD Lab 08: Container Image Scanning, Hadolint, and Pipeline Refactoring in GitLab CI

> The image is signed. The provenance is attached. Now what's actually inside it?

In the [previous post](https://medium.com/@pras-labs/cicd-lab-artifact-signing-slsa-provenance-in-gitlab-ci), I added cosign signing, SLSA v1 provenance attestation, and a `verify` stage to cryptographically link the image in the registry back to its source commit. By the end, you could prove the image was clean at scan time and that the artifact hadn't been swapped out after the pipeline ran.

But there was one gap I called out explicitly: `trivy fs` scans the source tree which includes your declared dependencies, your IaC, your language packages. It does not scan what actually ended up in the container image layers. Base image CVEs, packages pulled in by the distroless runtime, anything added by BuildKit but not visible in the source, none of that was covered.

This post closes that gap. I also took the opportunity to harden supply chain integrity across the whole pipeline by pinning every scanner image to a SHA256 digest, adding Dockerfile linting, and renaming the `build` stage to `container` to better reflect what it actually does.

You can follow along at [gitlab.com/pras-labs/ci-demo](https://gitlab.com/pras-labs/ci-demo/-/tree/feat/add-image-scanning-refactor-security).

---

### Why Digest Pinning Matters Now

The changes in this iteration were partly motivated by a real incident. In March 2026, the Aqua Security Trivy project was briefly compromised when there is a supply chain attack against the container images being distributed by the project. If your pipeline pulls `aquasec/trivy:latest` or even a pinned tag like `aquasec/trivy:0.69.3`, you are trusting that the tag still points to the same image it did when you last verified it. Tags are mutable. A compromised registry push can silently swap the image under a tag.

The fix is to pin by digest:

```dockerfile
# Before
FROM gcr.io/distroless/static-debian12:latest-amd64

# After
# Pin to digest for full reproducibility and supply chain integrity (tag latest-amd64 as reference).
# To update: TOKEN=$(curl -s "https://gcr.io/v2/token?scope=repository:distroless/static-debian12:pull&service=gcr.io" | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])"); curl -sI "https://gcr.io/v2/distroless/static-debian12/manifests/latest-amd64" -H "Authorization: Bearer $TOKEN" -H "Accept: application/vnd.oci.image.manifest.v1+json,application/vnd.docker.distribution.manifest.v2+json" | grep docker-content-digest
FROM gcr.io/distroless/static-debian12@sha256:340ba156c899ddac5ba57c5188b8e7cd56448eb7ee65b280574465eac2718ad2
```

I also applied this to every scanner image in the CI config that posed supply chain risk:

```yaml
# Every Trivy job now uses a pinned digest
image:
  name: docker.io/aquasec/trivy@sha256:bcc376de8d77cfe086a917230e818dc9f8528e3c852f7b1aff648949b6258d1c # pinned digest to mitigate aquasec/trivy breach March 2026
  entrypoint: [""]
```

And every image reference was updated to use the fully-qualified `docker.io/` registry prefix: `docker.io/library/busybox:1.37` instead of `busybox:1.37`, `docker.io/semgrep/semgrep:1.108.0` instead of `semgrep/semgrep:1.108.0`, and so on. This eliminates implicit registry resolution and makes the actual pull source explicit.

---

### Dockerfile Linting with Hadolint

Before scanning the image, it makes sense to lint the Dockerfile that builds it. I added `sast:hadolint` to the security stage:

```yaml
sast:hadolint:
  <<: *use-devenv
  stage: security
  script:
    - |
      hadolint \
        --failure-threshold error \
        --format sarif \
        --ignore DL3007 \
        Dockerfile | tee reports/sast/hadolint.sarif
    - |
      hadolint \
        --failure-threshold error \
        --format tty \
        --ignore DL3007 \
        Dockerfile
  extends: .on-security-scan
  allow_failure: false
  artifacts:
    when: always
    expire_in: 7 days
    paths:
      - reports/sast/hadolint.sarif
    reports:
      sast: reports/sast/hadolint.sarif
```

A few things worth explaining here:

- **`--failure-threshold error`:** hadolint has severity levels: error, warning, info, style. Setting the threshold to `error` means only rule violations at the error level fail the job. Warnings are surfaced in the output but don't block the pipeline.
- **`--ignore DL3007`:** DL3007 flags the use of `latest` tags in `FROM` instructions. We pin by digest in the Dockerfile, so the tag is present only as a human-readable reference. Ignoring this rule avoids a false positive.
- **Dual output:** the job runs hadolint twice: once to produce the SARIF artifact for GitLab's security dashboard, and once in TTY format for readable output directly in the CI log. SARIF output doesn't render well in a terminal, so the second run gives you something to actually look at when the job fails.
- **`allow_failure: false`:** Dockerfile errors are blocking. If hadolint finds a genuine error, the security gate does not pass.

`sast:hadolint` is added to the `security:gate` needs list, so it has to pass before the container stage can run.

---

### Renaming the Stage: build to container

The original pipeline had a `build` stage and a separate `verify` stage. The `build` stage produced the image; `verify` confirmed the signature and attestation. In practice, both stages operate on the container artifact, and the `verify` stage depended entirely on `build` completing first, there was no reason for it to be a separate stage.

I collapsed them into a single `container` stage and renamed all the jobs to match:

```yaml
stages:
  - preflight # Fast linting + format + secrets checks
  - security # SAST + SCA parallel scans
  - container # Container image build + image scan + signing
```

The job rename map:

| Before             | After                                  |
| ------------------ | -------------------------------------- |
| `build`            | `container:build`                      |
| `build:dind`       | `container:build:dind` (commented out) |
| `build:sign`       | `container:sign`                       |
| `build:provenance` | `container:provenance`                 |
| `verify:build`     | `container:verify`                     |

I also changed the `.on-container` rules. Previously, builds on MR and main branch were `when: manual` thus you had to click a button to run them. I changed these to `when: on_success`:

```yaml
.on-container:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: on_success
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: on_success
    - if: $CI_COMMIT_TAG
      when: on_success
```

The intent is to make container builds automatic. You still need security gate to pass first (`container:build` has `needs: [security:gate]`), so this doesn't skip anything, it just removes the manual click.

---

### Container Image Scanning with Trivy

The new `container:scan` job runs after `container:build` and scans the image that was just pushed to the registry:

```yaml
container:scan:
  stage: container
  image:
    name: docker.io/aquasec/trivy@sha256:bcc376de8d77cfe086a917230e818dc9f8528e3c852f7b1aff648949b6258d1c
    entrypoint: [""]
  needs:
    - container:build
    - cache:warm:trivy
  before_script:
    - mkdir -p reports/sca
    - mkdir -p ~/.docker
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"username\":\"$CI_REGISTRY_USER\",\"password\":\"$CI_REGISTRY_PASSWORD\"}}}" > ~/.docker/config.json
  script:
    - |
      trivy image \
        --cache-dir .cache/trivy \
        --scanners vuln \
        --severity HIGH,CRITICAL \
        --exit-code 1 \
        --format json \
        --output reports/sca/trivy-image.json \
        "$IMAGE_URL"
    # Human-readable summary
    - trivy image --cache-dir .cache/trivy --scanners vuln --severity HIGH,CRITICAL "$IMAGE_URL" || true
  cache:
    - <<: *cache-trivy
      policy: pull
  extends: .on-container
  allow_failure: false
```

The key difference from `sca:vuln`, which already runs `trivy fs` against the source tree, is the target. `trivy fs` scans your project's dependency manifests: `go.sum`, `package-lock.json`, `requirements.txt`. It finds CVEs in the packages you explicitly declared.

`trivy image` scans the actual layers of the built container image. It finds:

- CVEs in OS packages installed in the base image (in this case, `distroless/static-debian12`)
- Language packages that ended up in the image but weren't in the source manifests
- Anything the build process pulled in that `trivy fs` would never see

The `--exit-code 1` flag makes it blocking thus HIGH or CRITICAL findings in the image fail the job. The second `trivy image` invocation runs in TTY format without `--exit-code 1` so you get the human-readable table in the CI log regardless of whether the scan passed.

The `$IMAGE_URL` variable comes from `container:build` via a dotenv artifact. It contains the full registry URL with digest, so Trivy scans exactly the image that was just built and pushed.

---

### Image SBOM and Expanded Verification

`container:provenance` was previously limited to attaching the SLSA v1 provenance predicate. I extended it to also generate a CycloneDX SBOM for the image and attest it to the registry:

```yaml
container:provenance:
  # ...
  script:
    # Attach SLSA v1 provenance predicate from GitLab Runner auto-generated metadata
    - |
      cosign attest \
        --predicate <(jq '.predicate' artifacts-metadata.json) \
        --type slsaprovenance1 \
        --yes "$IMAGE_URL"
    # Generate image-level SBOM and attach to registry
    - |
      trivy image \
        --cache-dir .cache/trivy \
        --format cyclonedx \
        --output reports/sca/sbom-image.cdx.json \
        "$IMAGE_URL"
    - |
      cosign attest \
        --predicate reports/sca/sbom-image.cdx.json \
        --type cyclonedx \
        --yes "$IMAGE_URL"
```

This adds a second attestation to the image: a CycloneDX bill of materials covering the image layers, not just the source. The `sca:sbom` job already generates a filesystem-level SBOM; this one is image-level and lives in the registry as a verifiable attestation.

`container:verify` now checks all three:

```yaml
container:verify:
  script:
    # Verify image signature
    - |
      cosign verify \
        --certificate-identity-regexp "https://gitlab.com" \
        --certificate-oidc-issuer "https://gitlab.com" \
        "$IMAGE_URL"
    # Verify SLSA provenance attestation
    - |
      cosign verify-attestation \
        --type slsaprovenance1 \
        --certificate-identity-regexp "https://gitlab.com" \
        --certificate-oidc-issuer "https://gitlab.com" \
        "$IMAGE_URL"
    # Verify image SBOM attestation (CycloneDX)
    - |
      cosign verify-attestation \
        --type cyclonedx \
        --certificate-identity-regexp "https://gitlab.com" \
        --certificate-oidc-issuer "https://gitlab.com" \
        "$IMAGE_URL"
```

If any of the three verifications fail: signature invalid; provenance missing; SBOM attestation absent, the job fails.

---

### Container Gate

The final addition is `container:gate`, which requires both `container:scan` and `container:verify` to pass before the stage is considered complete:

```yaml
container:gate:
  stage: container
  image: docker.io/library/busybox:1.37
  needs:
    - container:scan
    - container:verify
  script:
    - echo "✅ Container security gate passed"
  extends: .on-container
  allow_failure: false
```

This is the same pattern used by `preflight:gate` and `security:gate`. A downstream deploy or release stage would declare `needs: [container:gate]` to ensure it only runs after both the scan and the verification have cleared.

---

### What the Pipeline Looks Like Now

```
preflight stage:
  preflight:secrets         (gitleaks)
  preflight:lint            (pre-commit hooks)
  preflight:commitlint      (commit message format)
  preflight:branch-name     (branch naming convention)
  preflight:gate            <- blocks security stage

security stage (all parallel):
  sast:semgrep              (multi-language SAST)
  sast:bandit               (Python SAST)
  sast:trufflehog           (deep secret scan)
  sast:hadolint             (Dockerfile lint)     <- NEW
  sast:checkov              (IaC SAST)
  sca:vuln                  (dependency CVEs)
  sca:license               (license compliance)
  sca:sbom                  (SBOM, main only)
  security:gate             <- blocks container stage

container stage (auto on MR/main/tags):
  container:build           (buildkit rootless, primary artifact)
  container:scan            (trivy image scan)    <- NEW
  container:sign            (cosign keyless sign)
  container:provenance      (SLSA v1 + CycloneDX SBOM) <- EXPANDED
  container:verify          (sig + provenance + SBOM)  <- EXPANDED
  container:gate            <- final container gate    <- NEW
```

---

### Conclusion

Here is what this iteration added:

- **Digest pinning:** all scanner images and the distroless base image pinned to SHA256 digests; all image refs use fully-qualified `docker.io/` prefixes; motivated by the March 2026 Trivy supply chain incident
- **`sast:hadolint`:** Dockerfile linting in the security stage; blocks on error-level findings; produces SARIF for GitLab's security dashboard
- **Stage rename:** `build` -> `container`; `verify` stage merged into `container`; jobs prefixed with `container:`; MR and main builds changed from `manual` to `on_success`
- **`container:scan`:** Trivy image scan against the pushed image; catches base image CVEs and layer-level vulnerabilities not visible to `trivy fs`; blocks on HIGH/CRITICAL findings
- **Image-level SBOM:** `container:provenance` now generates and attests a CycloneDX SBOM for the built image alongside the SLSA provenance
- **Expanded verification:** `container:verify` now checks signature, SLSA provenance attestation, and CycloneDX SBOM attestation
- **`container:gate`:** final gate requiring both scan and verify to pass before the stage completes

The security pipeline now covers the full artifact lifecycle: code is scanned before it builds, the image is scanned after it's built, the artifact is signed and attested, and nothing downstream can run until all of it passes.

---

### What's Next

With a complete scan-sign-attest-verify chain in place, the next step is putting the artifact to use. The pipeline currently ends at `container:gate`, there's no deploy or release stage yet. The next post will cover adding a deploy stage: pushing to an environment, using GitLab environments for deployment tracking, and wiring up the necessary secrets and access controls to make a real deployment work from CI.

Stay tuned for next stories with Pras' Labs: The Hands-On Engineer!

---

_Tags: GitLab CI, Container Security, Trivy, Hadolint, Supply Chain Security, DevSecOps, CI/CD, CycloneDX, SBOM, cosign_
