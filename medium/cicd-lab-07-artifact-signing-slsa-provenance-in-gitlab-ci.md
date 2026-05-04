# CI/CD Lab 07: Artifact Signing and SLSA Provenance in GitLab CI

> The security gate passed. The image is in the registry. But can you prove it?

In the [previous post](https://medium.com/@pras-labs/cicd-lab-fixing-cache-rules-and-trivy-warm-in-gitlab-ci), I fixed two silent bugs: `changes:` filters firing incorrectly on tag pipelines, and three SCA jobs racing to download the Trivy vulnerability database in parallel. Small fixes, but they matter, a flaky pipeline erodes trust in all the security checks running inside it.

With the pipeline behaving correctly, I turned to a different question: what happens after the build? A passing `security:gate` proves the source code was clean at scan time. It does not prove that the image sitting in the registry right now is the same one that came out of that pipeline. It does not prove which commit produced it, or that nobody pushed a replacement after the fact.

This post covers the supply chain layer: locking builds to deterministic inputs, signing the artifact with cosign using GitLab's OIDC identity, generating a SLSA v1 provenance attestation, and verifying both in a new `verify` stage. The result is a registry image you can cryptographically trace back to its source commit.

You can follow along at [gitlab.com/pras-labs/ci-demo](https://gitlab.com/pras-labs/ci-demo/-/tree/feat/reproducible-build-artifact-signing-slsa-provenance).

---

### Build Reproducibility Variables

Signing an artifact is only meaningful if the artifact is deterministic. A build that pulls different package versions on each run can produce a different image digest every time, which makes the signature useless for auditability, you cannot tell whether two digests came from the same inputs or not.

I added a set of CI variables to enforce deterministic behavior across the package ecosystems in use:

```yaml
variables:
  # Enforce build reproducibility rules
  ## Ensure npm installs are fully deterministic
  NPM_CONFIG_FUND: "false"
  NPM_CONFIG_AUDIT: "false"
  NPM_CONFIG_LEGACY_PEER_DEPS: "true" # avoid peer deps failures
  ## Rust/Go if used - pin exact versions too
  CARGO_NET_GIT_FETCH_WITH_CLI: "true" # cargo uses SSH, more reliable
  GOFLAGS: "-mod=readonly -modfile=go.mod" # forbid go.sum changes
```

- **`NPM_CONFIG_FUND: "false"`:** disables the funding nag on `npm install`, which can vary output and occasionally triggers network calls
- **`NPM_CONFIG_AUDIT: "false"`:** skips the auto-audit on install; we already run `sca:vuln` for this explicitly
- **`NPM_CONFIG_LEGACY_PEER_DEPS: "true"`:** prevents peer dependency resolution failures that can cause non-deterministic install errors depending on environment
- **`CARGO_NET_GIT_FETCH_WITH_CLI: "true"`:** routes Cargo's git fetches through the SSH CLI rather than the bundled `libgit2`, which is more reliable in restricted CI environments
- **`GOFLAGS: "-mod=readonly -modfile=go.mod"`:** forbids any job from modifying `go.sum` mid-build; if the sum is wrong, the build fails rather than silently fixing itself

These sit in the global `variables:` block, so they apply to every job. No per-job opt-in needed.

I also updated both cache keys to include `.mise.toml`:

```yaml
key:
  files:
    - package-lock.json
    - .mise.toml # bump version when tools change
  prefix: npm-v2
```

Previously, changing a tool version in `.mise.toml` without touching `package-lock.json` would silently reuse a stale cache built with the old toolchain. Adding `.mise.toml` to the key invalidates the cache when tools change.

---

### Extracting the Image Digest

cosign signs images by digest, `registry/image@sha256:abc123...`, not by tag. Tags are mutable; anyone with push access can overwrite `latest`. A digest is immutable: it is the SHA256 of the image manifest content. Signing by digest means the signature is bound to exactly one artifact.

BuildKit (used in the rootless build job) can emit a metadata file containing the digest of the pushed image. I added `--metadata-file metadata.json` to the `buildctl` invocation and then extracted the digest in a post-build step:

```yaml
script:
  - |
    buildctl build \
      --frontend dockerfile.v0 \
      --local context=. \
      --local dockerfile=. \
      --export-cache type=registry,ref=$CACHE_IMAGE \
      --import-cache type=registry,ref=$CACHE_IMAGE \
      --opt platform=linux/amd64 \
      --output type=image,name=$IMAGE_TAG,push=true \
      --metadata-file metadata.json
  # Extract the digest from the metadata file created by buildctl
  - |
    DIGEST=$(grep '"containerimage.digest"' metadata.json | sed 's/.*"containerimage.digest": *"\(sha256:[a-f0-9]*\)".*/\1/')
    echo "IMAGE_URL=$CI_REGISTRY_IMAGE@$DIGEST" | tee build.env
```

The `build.env` file is published as a `dotenv` artifact:

```yaml
artifacts:
  reports:
    dotenv: build.env
  paths:
    - artifacts-metadata.json
    - metadata.json
  expire_in: 1 hour
```

GitLab's `dotenv` artifact type injects the variables into downstream jobs automatically. Any job with `needs: [build]` gets `IMAGE_URL` in its environment without any extra scripting. I also enabled `RUNNER_GENERATE_ARTIFACTS_METADATA` and set `SLSA_PROVENANCE_SCHEMA_VERSION` to `v1` so GitLab generates its own runner-level metadata alongside the BuildKit metadata.

---

### Signing the Artifact with cosign (Keyless)

Traditional image signing requires managing a signing key: generating it, storing it securely (usually in a secret manager or Vault), rotating it, and distributing the public key to verifiers. That is real operational overhead. cosign's keyless mode skips all of it.

In keyless mode, cosign exchanges the CI job's OIDC token for a short-lived signing certificate from Fulcio (the Sigstore certificate authority). The certificate is bound to the OIDC identity: in this case, the GitLab job URL, the ref, and the project. The signature and certificate are recorded in Rekor, Sigstore's public transparency log. There is no private key to manage because no persistent key is ever created.

GitLab supports OIDC tokens via the `id_tokens` keyword:

```yaml
build:sign:
  <<: *use-devenv
  stage: build
  needs:
    - build
  id_tokens:
    SIGSTORE_ID_TOKEN:
      aud: sigstore
  before_script:
    - cosign login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - cosign sign --yes $IMAGE_URL
  extends: .on-build
```

- **`id_tokens.SIGSTORE_ID_TOKEN.aud: sigstore`:** GitLab generates a JWT with `sigstore` as the audience. cosign picks this up automatically from the environment variable name.
- **`cosign sign --yes $IMAGE_URL`:** signs the image at the digest stored in `IMAGE_URL` from the upstream `build` job's dotenv artifact. `--yes` skips the interactive confirmation prompt.
- **`needs: [build]`:** waits for the build job and inherits its `IMAGE_URL` variable.

The `build:sign` job needs to run after `build` but is otherwise independent of `build:provenance`. Both can run in parallel.

---

### SLSA Provenance Attestation

A signature proves the artifact has not been tampered with since it was signed. A provenance attestation goes further: it describes _how_ the artifact was built: which repository, which commit, which builder, which pipeline. This is what SLSA (Supply-chain Levels for Software Artifacts) provenance is.

GitLab's runner metadata (`artifacts-metadata.json`) contains the build context in a format that maps to the SLSA v1 predicate schema. The attestation job attaches this as a signed statement to the image:

```yaml
build:provenance:
  <<: *use-devenv
  stage: build
  needs:
    - job: build
      artifacts: true
  id_tokens:
    SIGSTORE_ID_TOKEN:
      aud: sigstore
  before_script:
    - cosign login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - |
      cosign attest \
        --predicate <(jq '.predicate' artifacts-metadata.json) \
        --type slsaprovenance1 \
        --yes "$IMAGE_URL"
  extends: .on-build
```

- **`needs: [job: build, artifacts: true]`:** explicitly requests artifact download, which gives the job access to `artifacts-metadata.json` and `build.env`
- **`--predicate <(jq '.predicate' artifacts-metadata.json)`:** extracts the predicate field from the GitLab-generated metadata and pipes it directly to cosign via process substitution; no intermediate temp file
- **`--type slsaprovenance1`:** specifies the SLSA provenance v1 schema, which is the current stable version

The attestation is stored in the registry as an OCI artifact attached to the signed image digest. Verifiers can pull and inspect it without any separate storage.

---

### `.on-build` Rules Template

All three build-stage jobs (`build`, `build:sign`, `build:provenance`) use the same trigger conditions: manual on MR and default branch, automatic on tags. I extracted this into a shared template rather than repeating the `rules:` block three times:

```yaml
.on-build:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: manual
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
    - if: $CI_COMMIT_TAG
      when: on_success
```

Any job that `extends: .on-build` gets these rules. Tags trigger automatic builds and signing; MR and default branch builds are manual to avoid burning registry storage on every commit.

---

### Verification Stage

Signing and attesting are only useful if verification happens. The `verify` stage runs after the entire build stage completes and confirms that the image in the registry has a valid signature and a valid provenance attestation:

```yaml
verify:build:
  <<: *use-devenv
  stage: verify
  needs:
    - build
    - build:sign
    - build:provenance
  before_script:
    - cosign login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - |
      cosign verify \
        --certificate-identity-regexp "https://gitlab.com" \
        --certificate-oidc-issuer "https://gitlab.com" \
        "$IMAGE_URL"
    - |
      cosign verify-attestation \
        --type slsaprovenance1 \
        --certificate-identity-regexp "https://gitlab.com" \
        --certificate-oidc-issuer "https://gitlab.com" \
        "$IMAGE_URL"
  extends: .on-build
```

- **`--certificate-identity-regexp "https://gitlab.com"`:** accepts certificates issued for any GitLab job URL. In a stricter setup you'd pin this to the exact project path.
- **`--certificate-oidc-issuer "https://gitlab.com"`:** verifies the certificate was issued by GitLab's OIDC provider, not some other Sigstore-compatible issuer.
- **`cosign verify-attestation --type slsaprovenance1`:** specifically checks for the SLSA v1 attestation, not just any attestation.

If either check fails: signature missing; certificate identity mismatch; attestation absent, the `verify` stage fails and the pipeline ends there. An image that cannot be verified does not get deployed.

The `needs:` list includes all three upstream jobs, so verification only runs once signing and attestation have both completed.

---

### Dockerfile Healthcheck (Distroless Trick)

Unrelated to signing but worth covering since it landed in the same branch: I added a `HEALTHCHECK` to the Dockerfile. The image uses `gcr.io/distroless/static-debian12` as the final stage: no shell, no package manager, no `curl`, no `wget`. Standard healthcheck techniques do not work.

The clean solution: copy a static `wget` binary from `busybox:musl` at build time.

```dockerfile
# Healthcheck
# distroless has no shell/curl/wget, so copy a static wget binary from busybox
COPY --from=busybox:musl /bin/wget /wget
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD ["/wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/healthz"]
```

`busybox:musl` is statically linked thus no libc dependencies. The `wget` binary runs fine inside the distroless container because it does not need anything from the host filesystem. `--spider` does a HEAD-like request without downloading the body, which is all a healthcheck needs. The binary sits at `/wget` to avoid any naming conflicts.

---

### Checkov Pre-Push Hook

One smaller change: the Checkov IaC pre-push hook that was commented out in `.pre-commit-config.yaml` is now active:

```yaml
- id: sast-checkov-prepush
  name: Checkov IaC scan
  stages: [pre-push]
  language: system
  entry: bash -c 'checkov -d . --framework dockerfile,kubernetes,gitlab_ci
    --soft-fail-on MEDIUM --hard-fail-on HIGH,CRITICAL --compact'
  pass_filenames: false
```

This runs Checkov locally on every `git push` across Dockerfile, Kubernetes manifests, and GitLab CI YAML. MEDIUM findings are soft failures (warning only), HIGH and CRITICAL block the push. The CI pipeline already runs Checkov in `sast:checkov`, this pre-push hook catches misconfigurations before they reach the pipeline at all.

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
  sast:secrets:trufflehog   (deep secret scan)
  sast:checkov              (IaC SAST)
  sca:vuln                  (dependency CVEs)
  sca:license               (license compliance)
  sca:sbom                  (SBOM, main only)
  security:gate             <- blocks build stage

build stage (manual on MR/main, auto on tags):
  build:dind                (docker build + push, DinD)
  build                     (buildkit rootless, primary artifact)  <- RENAMED
  build:sign                (cosign keyless sign)                  <- NEW
  build:provenance          (cosign attest SLSA v1)                <- NEW

verify stage (manual on MR/main, auto on tags):
  verify:build              (cosign verify + verify-attestation)   <- NEW
```

---

### Conclusion

Here is what this iteration added:

- **cosign:** keyless signing via GitLab OIDC + Sigstore Fulcio; no key management required
- **`build:sign`:** signs the image digest after push; certificate bound to GitLab job identity
- **`build:provenance`:** attaches a SLSA v1 provenance attestation using GitLab runner metadata as the predicate
- **`verify:build`:** new `verify` stage that validates signature and attestation before the pipeline ends; failures block promotion
- **BuildKit metadata extraction:** `--metadata-file metadata.json` captures the image digest; `dotenv` artifact passes `IMAGE_URL` to downstream jobs
- **Reproducibility variables:** `GOFLAGS`, `NPM_CONFIG_*`, `CARGO_NET_*` enforce deterministic inputs
- **Cache key improvements:** `.mise.toml` added to cache keys so tool version changes bust stale caches
- **Distroless healthcheck:** static `wget` from `busybox:musl` enables `HEALTHCHECK` in a shell-free image
- **Checkov pre-push hook:** IaC scanning before push, blocking on HIGH/CRITICAL findings

A passing `security:gate` proves the code was clean. A valid `verify:build` proves the artifact in the registry is the one the pipeline produced, and that you can trace it back to the exact commit and job that built it.

---

### What's Next

With signed and attested images in the registry, the next logical step is Trivy image scanning, running Trivy against the built container image itself, not just the source dependencies. Source-level SCA finds what you declared; image scanning finds what actually ended up in the layers, including anything pulled in through the base image. That is the last gap in the security pipeline before we can call the full scan-to-deploy chain complete.

Stay tuned for next stories with Pras' Labs: The Hands-On Engineer!

---

_Tags: GitLab CI, Supply Chain Security, SLSA, cosign, Sigstore, Provenance, DevSecOps, CI/CD, Artifact Signing, Distroless_
