# Social Media Posts: CI/CD Lab #3 — SAST and Static Analysis in Gitlab CI

Source post: `medium/cicd-lab-03-sast-static-analysis-in-gitlab-ci.md`

---

## 𝕏 Twitter/X

**Post 1** (hook):
```
Lint and secret scans won't catch SQL injection, IaC misconfigs, or a working AWS key buried in git history.

I added a SAST stage to my GitLab CI pipeline: Semgrep, Trufflehog, Bandit, Checkov — four scanners, one gate. Build doesn't run until all four pass.
```
*(267 chars)*

**Post 2** (CTA):
```
New CI/CD Lab post: 4 scanners, diff-aware MR scanning, SARIF wired to GitLab Security tab, and `make sast-diff` so you can mirror the MR pipeline locally.

Full post: [link]

#GitLabCI #SAST #DevSecOps #CI #DevOps #HandsOnEngineer
```
*(255 chars)*

---

## 💼 LinkedIn

```
Most CI pipelines I've seen catch bad formatting and leaked secrets. They miss the harder stuff: an injection pattern in application code, a privileged: true in a Kubernetes manifest, or a working API key that gitleaks flagged but couldn't confirm was live.

In the latest CI/CD Lab post, I added a dedicated SAST stage to my GitLab CI pipeline. Four scanners, a gate that blocks build until all pass, and `make` targets that mirror the pipeline locally.

What's covered:

• Semgrep with 5 rule packs (OWASP Top 10, secrets, CI, Dockerfile, Kubernetes) — diff-aware on MRs so it only scans changed files
• Trufflehog alongside gitleaks: gitleaks flags candidates, Trufflehog verifies them against live providers with --only-verified
• Bandit for Python with a conditional guard (no-ops cleanly if there are no .py files)
• Checkov for IaC: soft-fail on MEDIUM, hard-fail on HIGH/CRITICAL, SARIF wired into GitLab's Security tab

The pipeline restructure matters too: build is now the last stage and manual. Docker-in-Docker minutes don't burn until the full preflight + SAST chain passes.

If you're building a secure pipeline and want to see the actual GitLab CI YAML, not pseudocode, this post has the real config.

Repo: https://gitlab.com/pras-labs/ci-demo/-/tree/feat/sast-static-analysis

#GitLabCI #SAST #DevSecOps #DevOps #ShiftLeft #Security #CI
```

---

## ✉️ Email Subject Lines

1. `Why does my pipeline pass lint but ship vulnerable code?`
2. `I wired 4 SAST scanners into GitLab CI — here's the full config`
3. `SAST in GitLab CI: Semgrep, Trufflehog, Bandit, Checkov (with real YAML)`
4. `CI/CD Lab #3: SAST and Static Analysis in GitLab CI`

---

## ☁️ Bluesky

**Post 1**:
```
Preflight checks catch hygiene. SAST catches what's actually dangerous.

New CI/CD Lab post: I added Semgrep, Trufflehog, Bandit, and Checkov to GitLab CI. Build stage blocked until all four pass. Semgrep scans only changed files on MRs.
```
*(264 chars)*

**Post 2**:
```
Also covers: Trufflehog --only-verified (only fails on working keys), Checkov soft/hard severity split, SARIF wired to GitLab Security tab, and make sast-diff to mirror MR scanning locally.

Full post + repo: [link]
```
*(213 chars)*

---

## 🧵 Threads

**Post 1**:
```
I added a full SAST stage to my GitLab CI pipeline. Four scanners: Semgrep, Trufflehog, Bandit, Checkov. One gate. Build doesn't run until all four pass.

Preflight lint catches style and secrets on the surface. SAST catches injection patterns, IaC misconfigs, and verified live credentials buried in git history. Different class of problem.
```
*(345 chars)*

**Post 2**:
```
What's in the post:

• Semgrep with OWASP Top 10 + Dockerfile + Kubernetes packs — diff-aware on MRs (scans changed files only, full scan on default branch)
• Trufflehog with --only-verified alongside gitleaks — validates candidates against live providers
• Bandit for Python (conditional, no-ops cleanly with no .py files)
• Checkov: soft-fail MEDIUM, hard-fail HIGH/CRITICAL, SARIF feeds GitLab Security tab
```
*(394 chars)*

**Post 3**:
```
Pipeline restructure: build is now the last stage and manual. No Docker-in-Docker minutes until the entire preflight + SAST chain passes.

Every scanner also has a matching `make` target. `make sast-diff` mirrors the MR changed-files behavior so you can verify before pushing.

Full post + real YAML (not pseudocode): [link]
Repo: https://gitlab.com/pras-labs/ci-demo/-/tree/feat/sast-static-analysis
```
*(421 chars)*

---

## 📘 Facebook

```
Just published the next CI/CD Lab post: adding a SAST stage to GitLab CI with four scanners (Semgrep, Trufflehog, Bandit, Checkov).

The thing that surprised me most while building this: gitleaks and Trufflehog are not redundant. Gitleaks flags candidates fast. Trufflehog verifies them — it actually calls out to AWS/GitHub/Slack to check if the key still works. That's a different threat model.

Full config is in the post, including the diff-aware MR scanning and how to mirror the pipeline locally with make targets.

If you've set up something similar, curious what scanners you're using — drop it in the comments.

[link]
```

---

*Hashtags chosen: `#GitLabCI` and `#SAST` / `#DevSecOps` are the most specific to the post content. `#ShiftLeft` on LinkedIn where longer tag lists are normal. `#HandsOnEngineer` on X to build series continuity. Skipped broad tags like `#programming` — too generic for this audience.*
