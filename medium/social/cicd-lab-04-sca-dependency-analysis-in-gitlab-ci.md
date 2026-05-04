# Social Posts: CI/CD Lab: Software Composition Analysis with Trivy in GitLab CI

---

## 𝕏 Twitter/X

**Post 1** (271 chars):
```
SAST scans code I wrote. It doesn't scan the 847 npm packages I pulled in.

Added SCA to my GitLab CI pipeline with Trivy: dependency CVEs, license compliance, and a CycloneDX SBOM. Also cleaned up a pile of copy-pasted rules: blocks while I was in there.
```

**Post 2** (242 chars):
```
New CI/CD Lab post: Software Composition Analysis with Trivy in GitLab CI

Three new jobs (sca:vuln, sca:license, sca:sbom), a unified security:gate, and pipeline cleanup that cut repeated YAML significantly.

Full walkthrough: [link]

#GitLabCI #DevSecOps #SCA #CI #DevOps #HandsOnEngineer
```

---

## 💼 LinkedIn

```
Most pipelines scan the code their team wrote. Very few scan the code they pulled in from the internet.

That's the gap SCA fills. In the latest CI/CD Lab post, I added Software Composition Analysis to the GitLab CI pipeline using Trivy — the same tool already doing Dockerfile scanning in the devcontainer.

Here's what's in the post:

• sca:vuln — Trivy fs scan against package-lock.json and go.sum, blocking the pipeline on HIGH/CRITICAL CVEs, with a dependency_scanning artifact for GitLab's Security tab
• sca:license — License compliance check that warns (but doesn't block) when a restricted or forbidden license shows up in the dependency tree
• sca:sbom — CycloneDX Software Bill of Materials generated on the default branch; 30-day retention for triage when a new CVE drops
• security:gate — Renamed from sast:gate, now requires both SAST and SCA to pass before any build job runs

I also did a pipeline cleanup pass: workflow: rules to stop duplicate pipelines, a default: block for interruptible/timeout/retry, and a reusable .on-mr-or-default template that replaced the same two-line rules: block copy-pasted into a dozen jobs.

If you're running SAST but not SCA, you have coverage over your first-party code and a blind spot over everything you installed. This post shows how to close that gap with one tool you may already have.

Full post: [link]

#GitLabCI #DevSecOps #SCA #SoftwareCompositionAnalysis #Trivy #SBOM #CycloneDX #SupplyChainSecurity #DevOps #CI
```

---

## ✉️ Email Subject Lines

1. `What are your npm packages hiding?`
2. `I added dependency scanning to my GitLab CI pipeline (here's exactly how)`
3. `SCA with Trivy: vuln scan, license check, and SBOM in one pipeline stage`
4. `CI/CD Lab #4: Software Composition Analysis with Trivy`

---

## ☁️ Bluesky

**Post 1** (298 chars):
```
SAST scans the code you wrote.
SCA scans the code you pulled in.

New CI/CD Lab post: I added Trivy-based SCA to GitLab CI — dependency CVE scanning, license compliance, and CycloneDX SBOM generation. All running parallel to SAST in the same stage.
```

**Post 2** (280 chars):
```
Also in the post: pipeline cleanup with workflow: rules (no more duplicate pipelines), a default: block for interruptible/retry, and a reusable .on-mr-or-default template.

Full walkthrough + all YAML: [link]

#DevSecOps #GitLabCI #SCA #Trivy
```

---

## 🧵 Threads

**Post 1** (468 chars):
```
Four SAST scanners in my GitLab CI pipeline. None of them scan npm packages or Go modules for known CVEs.

That's what SCA is for. I added three new jobs using Trivy: one for dependency vulnerabilities, one for license compliance, and one that generates a CycloneDX SBOM on every merge to main.

All of it runs in parallel with SAST. The security:gate job now blocks builds until both SAST and SCA are clean.
```

**Post 2** (448 chars):
```
What's in the new CI/CD Lab post:

• sca:vuln — Trivy fs mode, HIGH/CRITICAL CVEs, exit-code 1, JSON artifact for GitLab's Security tab
• sca:license — restricted/forbidden license detection; allow_failure: true so it warns without blocking
• sca:sbom — CycloneDX bill of materials, main branch only, 30-day retention
• Pipeline cleanup: workflow: rules, default: block, reusable .on-mr-or-default template
```

**Post 3** (312 chars):
```
If you're running SAST but not SCA, you're scanning the code you wrote and ignoring everything you installed.

This post shows how to close that gap with Trivy, which you may already have in your devcontainer.

Full walkthrough: [link]

#GitLabCI #DevSecOps #SCA #Trivy #SupplyChainSecurity
```

---

## 📘 Facebook

```
Just published part 4 of the CI/CD Lab series: adding Software Composition Analysis with Trivy to a GitLab CI pipeline.

The thing that prompted it — SAST tools like Semgrep and Bandit scan code you wrote, but they don't know about CVEs in your npm or Go dependencies. This post walks through exactly how I wired up dependency scanning, license compliance, and SBOM generation, all running in parallel with the existing SAST stage.

Also did a pipeline cleanup pass that I've been meaning to do for a while (goodbye, copy-pasted rules: blocks).

Anyone else running both SAST and SCA, or just one? Curious what tooling people are using for dependency scanning outside GitLab Ultimate.

Full post: [link]
```

---

_Hashtags: #DevSecOps #SCA #Trivy #GitLabCI #SBOM #CycloneDX #SupplyChainSecurity — topic-specific to post tools; #CI #DevOps #HandsOnEngineer for series continuity and discoverability._
