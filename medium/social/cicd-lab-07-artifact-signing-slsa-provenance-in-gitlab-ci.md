# Social Posts: CI/CD Lab 07 — Artifact Signing and SLSA Provenance in GitLab CI

---

## 𝕏 Twitter/X

**Post 1** (hook):
```
The security gate passed. The image is in the registry. But can you prove it wasn't replaced afterward?

In CI/CD Lab 07 I added cosign keyless signing + SLSA v1 provenance attestation to GitLab CI — no key management required, just GitLab's OIDC token.
```
_(277 chars)_

**Post 2** (CTA):
```
What's covered:
→ BuildKit digest extraction via --metadata-file
→ Keyless cosign sign with GitLab OIDC id_tokens
→ cosign attest with SLSA provenance v1
→ verify stage that blocks if signature is missing
→ Static wget in distroless for HEALTHCHECK

#CI #DevOps #Engineering #HandsOnEngineer #SupplyChainSecurity #SLSA
```
_(271 chars)_

---

## 💼 LinkedIn

```
A passing security scan proves your code was clean at scan time. It does not prove the image sitting in your registry right now came from that pipeline.

That gap is what artifact signing and provenance attestation close.

In the latest CI/CD Lab post I added the full supply chain layer to a GitLab CI pipeline:

• cosign keyless signing — GitLab OIDC token exchanges for a short-lived Fulcio certificate, signature recorded in Rekor. No private key to manage, ever.
• BuildKit metadata extraction — --metadata-file captures the exact image digest so signing is bound to one immutable artifact, not a mutable tag
• SLSA v1 provenance attestation — GitLab runner metadata becomes the predicate; cosign attest stores it as an OCI artifact attached to the digest
• verify stage — cosign verify + cosign verify-attestation run after the full build stage; missing or invalid signatures block the pipeline
• Bonus: static wget from busybox:musl enables HEALTHCHECK in a distroless container that has no shell

The result: every image in the registry is cryptographically traceable back to the exact commit and pipeline job that produced it. Verification is part of the pipeline, not an afterthought.

Full walkthrough with all the YAML: https://medium.com/@pras-labs/cicd-lab-artifact-signing-slsa-provenance-in-gitlab-ci

#GitLabCI #SupplyChainSecurity #SLSA #cosign #Sigstore #DevSecOps #CICD #DevOps
```
_(1,198 chars)_

---

## ✉️ Email Subject Lines

1. **Curiosity:** `Can you prove that image in your registry came from your pipeline?`
2. **How I:** `How I added keyless artifact signing and SLSA provenance to GitLab CI`
3. **Direct:** `GitLab CI: cosign signing + SLSA v1 attestation + verify stage (full walkthrough)`
4. **Series:** `CI/CD Lab 07: Artifact Signing and SLSA Provenance in GitLab CI`

---

## ☁️ Bluesky

**Post 1** (hook):
```
Security gate passed. Image in registry. But can you prove nobody replaced it after the push?

CI/CD Lab 07: I added cosign keyless signing + SLSA v1 provenance attestation to GitLab CI using id_tokens — no keys to manage, signature bound to the exact image digest.
```
_(272 chars)_

**Post 2** (details + CTA):
```
Also covered: BuildKit --metadata-file for digest extraction, dotenv artifacts to pass IMAGE_URL downstream, a verify stage that fails if signatures are missing, and a distroless HEALTHCHECK trick using static wget from busybox:musl.

Full post: https://medium.com/@pras-labs/cicd-lab-artifact-signing-slsa-provenance-in-gitlab-ci
```
_(296 chars)_

---

## 🧵 Threads

**Post 1:**
```
CI/CD Lab 07 is up.

The security gate proves your code was clean. It does not prove the image in your registry is the one the pipeline built.

I added cosign keyless signing, SLSA v1 provenance attestation, and a verify stage to GitLab CI. No private keys involved — just GitLab's OIDC token and Sigstore's Fulcio CA.
```
_(327 chars)_

**Post 2:**
```
What the post covers:

• BuildKit --metadata-file to capture the image digest (sign by digest, not tag — tags are mutable)
• cosign sign with GitLab id_tokens — short-lived Fulcio cert, recorded in Rekor
• cosign attest with GitLab runner metadata as the SLSA v1 predicate
• verify stage: cosign verify + cosign verify-attestation; pipeline fails if either is missing
```
_(374 chars)_

**Post 3:**
```
Also: a distroless HEALTHCHECK trick — copy static wget from busybox:musl at build time. No shell, no libc dependencies, works inside gcr.io/distroless/static.

Full walkthrough with all the YAML at the link in my profile.

Series: CI/CD Lab — building a real DevSecOps pipeline from scratch, one commit at a time.
```
_(317 chars)_

---

## 📘 Facebook

```
Just published CI/CD Lab 07 — this one's about what happens after the build.

A passing security scan tells you the code was clean. It doesn't tell you whether the image in your registry is actually the one your pipeline built. Artifact signing and provenance attestation close that gap.

I walk through adding cosign keyless signing and SLSA v1 attestation to a GitLab CI pipeline — no key management, just OIDC tokens and Sigstore. There's also a neat trick for adding HEALTHCHECK to a distroless container that has no shell or curl.

Who else has been thinking about supply chain security in their CI pipelines? What approach are you using?
```
_(621 chars)_

---

**Hashtag rationale:** `#SupplyChainSecurity` and `#SLSA` chosen as topic-specific tags — both are high-signal for the audience that cares about this content. `#cosign` and `#Sigstore` added for LinkedIn where tool-specific tags aid discoverability. Kept `#HandsOnEngineer` on X for series continuity.
