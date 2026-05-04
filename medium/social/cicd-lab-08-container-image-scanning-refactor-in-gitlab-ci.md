## 𝕏 Twitter/X

**Post 1** (278 chars):
```
trivy fs scans your source deps. It doesn't scan what actually ended up in the image layers.

Lab 08: I added trivy image scanning to GitLab CI — plus hadolint, digest pinning (motivated by the Trivy breach), and a full container gate.
```

**Post 2** (241 chars):
```
What's in the post:
→ Pin scanner images to SHA256 digests
→ sast:hadolint for Dockerfile linting
→ container:scan with trivy image
→ Image-level CycloneDX SBOM via cosign attest

Full walkthrough: [link]

#CI #DevOps #Engineering #HandsOnEngineer #SupplyChainSecurity #DevSecOps
```

---

## 💼 LinkedIn

```
Your pipeline signs the image. It attests the provenance. But does it scan what's actually inside the container?

trivy fs scans declared dependencies. It does not scan the base image layers, OS packages, or anything BuildKit pulled in that never appeared in your manifests. That's the gap Lab 08 closes.

Here's what this post covers:

• container:scan — Trivy image scanning against the pushed image, blocking on HIGH/CRITICAL CVEs, using the exact digest produced by the build
• sast:hadolint — Dockerfile linting in the security stage, with SARIF output for GitLab's dashboard and a DL3007 ignore to avoid false positives on digest-pinned FROM instructions
• Digest pinning — all scanner images (including Trivy itself) pinned to SHA256 digests after the March 2026 Trivy supply chain incident; fully-qualified docker.io/ refs everywhere
• Pipeline refactor — build stage renamed to container, verify stage merged in, manual triggers changed to on_success, jobs prefixed with container:

By the end: code is scanned before it builds, the image is scanned after it's built, the artifact is signed and attested with both SLSA provenance and a CycloneDX SBOM, and nothing downstream runs until all of it passes.

Full post with the actual YAML: [link]

#GitLabCI #DevSecOps #SupplyChainSecurity #ContainerSecurity #CI #DevOps #HandsOnEngineer
```

---

## ✉️ Email Subject Lines

1. `What does trivy fs miss that trivy image catches?`
2. `I added container image scanning (and Hadolint) to the GitLab CI pipeline`
3. `GitLab CI: Trivy image scan, Hadolint, digest pinning — Lab 08`
4. `CI/CD Lab #8: Container image scanning and pipeline refactoring in GitLab CI`

---

## ☁️ Bluesky

**Post 1** (298 chars):
```
trivy fs scans your source deps. trivy image scans what's actually in the layers — base image CVEs, OS packages, anything your build pulled in.

Lab 08 adds container:scan to the GitLab CI pipeline. Plus hadolint and SHA256 digest pinning for every scanner image.
```

**Post 2** (289 chars):
```
Also in this post:
→ sast:hadolint in the security stage
→ build stage renamed to container
→ Image-level CycloneDX SBOM via cosign attest
→ container:gate blocking until scan + verify pass

Motivated partly by the March 2026 Trivy breach. Full YAML walkthrough: [link]
```

---

## 🧵 Threads

**Post 1** (492 chars):
```
trivy fs scans your declared dependencies. It does not scan what ended up in the container image layers.

Base image CVEs, OS packages from distroless, anything BuildKit pulled in — none of that shows up in a filesystem scan.

Lab 08 closes that gap: I added trivy image scanning to the GitLab CI pipeline, running against the exact image digest that was just built and pushed.
```

**Post 2** (487 chars):
```
What else changed in this iteration:

• sast:hadolint — Dockerfile linting in the security stage, SARIF output for the dashboard, blocks on error-level findings
• Digest pinning — all Trivy jobs now use a SHA256 digest instead of a tag (motivated by the March 2026 Trivy supply chain breach); same for distroless base image and busybox
• build → container stage — verify stage merged in, jobs renamed, manual triggers changed to on_success
```

**Post 3** (312 chars):
```
By the end: code scanned before build, image scanned after, artifact signed + attested with SLSA provenance and a CycloneDX SBOM, nothing downstream runs until container:gate passes.

Full YAML walkthrough in the post: [link]

CI/CD Lab series continues.
```

---

## 📘 Facebook

```
Lab 08 of the CI/CD Lab series is up: container image scanning, Dockerfile linting, and supply chain hardening in GitLab CI.

The interesting bit: trivy fs (which I already had) scans your source dependencies. It doesn't scan what's actually in the image layers — base image packages, OS CVEs, anything the build pulled in. This post adds trivy image to close that gap, plus hadolint for Dockerfile checks and SHA256 digest pinning for every scanner image (after the March 2026 Trivy breach made that feel urgent).

Full write-up with the actual CI YAML: [link]

Anyone else pinning scanner images by digest now, or still trusting tags?
```

---

_Hashtags chosen: `#SupplyChainSecurity` and `#ContainerSecurity` added alongside `#DevSecOps` — this post is specifically about image-layer CVEs and the Trivy breach, so supply chain and container tags are more specific than generic `#Security`. `#HandsOnEngineer` kept as series brand marker._
