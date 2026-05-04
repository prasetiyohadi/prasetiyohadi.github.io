# Social Posts: CI/CD Lab 05 — Refactoring GitLab CI Rules and Stage Naming

Source post: `medium/cicd-lab-05-refactoring-gitlab-ci-rules-and-stage-naming.md`

---

## 𝕏 Twitter/X

**Post 1** (277 chars)
One GitLab CI rule template applied to every job — lint, SAST, SBOM, build. It worked. It was also lying. I split it into 3 purpose-specific templates and wired up auto-build on git tags. Each job's trigger scope now matches what it actually does.

**Post 2** (268 chars)
Full breakdown: `.on-mr-only` for quality gates, `.on-security-scan` for Semgrep/Trivy/Bandit/etc, `.on-release` for SBOM. Plus optional `needs:` deps and tag auto-build. CI/CD Lab 05 is up. Link in bio. #GitLabCI #CI #DevOps #Engineering #HandsOnEngineer #DevSecOps

---

## 💼 LinkedIn

I had one GitLab CI rule template applied to every job in the pipeline. It ran lint checks on the default branch after a merge — where they catch nothing. It skipped SBOM generation on release tags — where you actually need it. It worked, but it was wrong.

CI/CD Lab 05 is about fixing that: splitting one overloaded template into three that actually say what they mean.

What the post covers:
- Renaming the `sast` stage to `security` — accurate now that Trivy SCA and SAST both live there
- `.on-mr-only` for quality-only jobs (lint, commitlint, branch naming) that have no business running post-merge
- `.on-security-scan` for all SAST/SCA jobs (Semgrep, Trufflehog, Checkov, Bandit, Trivy) — MR, main, and tags, so a release tag can never skip a security gate
- `.on-release` for SBOM generation — main and tags only, not every feature branch
- Optional `needs:` dependencies so the preflight gate doesn't fail on non-MR pipelines
- Auto-build on `$CI_COMMIT_TAG` — push a tag, security gate runs, image builds automatically

If your pipeline has a single rule template doing everything, this is the refactor. No new tools, just YAML that tells the truth.

Full post: [link in comments]

#GitLabCI #DevSecOps #CICD #DevOps #Engineering #SAST #SCA #PipelineAsCode

---

## ✉️ Email Subject Lines

1. Why was my lint check running after every merge to main?
2. How I replaced one GitLab CI rule template with three that actually make sense
3. GitLab CI: split your rule templates before they lie to you
4. CI/CD Lab 05: refactoring GitLab CI rules and stage naming

---

## ☁️ Bluesky

**Post 1** (298 chars)
One rule template applied to every GitLab CI job — lint, Semgrep, Trivy, SBOM. It worked until I noticed lint was running post-merge where it catches nothing, and SBOM was skipping release tags where you need it most. The rule wasn't wrong, it just didn't know what each job was for.

**Post 2** (292 chars)
CI/CD Lab 05: split into `.on-mr-only`, `.on-security-scan`, `.on-release`. Fixed the preflight gate with optional deps. Added tag auto-build so pushing a release tag triggers security scan + image build with no manual click. Full post up now — link in bio.

---

## 🧵 Threads

**Post 1** (496 chars)
I had one GitLab CI rule template doing everything. Lint checks ran on the default branch after merges — useless there. SBOM generation skipped release tags — wrong. Security scans missed tag pipelines entirely — dangerous. The template was `.on-mr-or-default` and it was applied uniformly to every single job. It worked. It was semantically wrong. CI/CD Lab 05 is about replacing it with three templates that each say exactly what they mean.

**Post 2** (489 chars)
Here is what changed:
- `.on-mr-only` — lint, commitlint, branch-name. Quality gates only run when code is being proposed, not after it lands.
- `.on-security-scan` — Semgrep, Trufflehog, Checkov, Bandit, Trivy vuln + license. Runs on MR, main, and tags. No release skips the security gate.
- `.on-release` — SBOM generation on main and tags. Not every feature branch needs a bill of materials.
Also: `sast` stage renamed to `security`, optional `needs:` deps, and auto-build on git tags.

**Post 3** (321 chars)
Push a tag → security stage runs → image builds automatically. No manual gate, no forgotten click. The full post walks through every change with the YAML. CI/CD Lab 05 is up on Medium. Link in bio.

---

## 📘 Facebook

Been working through a GitLab CI cleanup this week — one rule template was applied to every job in the pipeline, from lint checks to security scans to SBOM generation. It worked, but each job was triggering in contexts where it made no sense.

CI/CD Lab 05 covers splitting it into three focused templates, fixing the preflight gate for non-MR pipelines, and wiring up automatic image builds when you push a release tag.

If you've ever looked at a pipeline and thought "why is that job running here?" — this one's for you. What's the most overloaded YAML pattern you've had to clean up?

---

*Hashtag choices: `#GitLabCI` and `#DevSecOps` added because the post is specifically about GitLab rule syntax and the security-scan template covers SAST/SCA tools — more precise than generic `#YAML`. `#SCA` and `#SAST` on LinkedIn where longer-form search matters. Facebook kept hashtag-free to match its community tone.*
