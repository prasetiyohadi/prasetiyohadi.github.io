# Social Posts: CI/CD Lab 06 — Fixing GitLab CI Cache Rules and Pre-Warming the Trivy DB

Source post: `medium/cicd-lab-06-fixing-cache-rules-and-trivy-warm-in-gitlab-ci.md`

---

## 𝕏 Twitter/X

**Post 1** (267 chars):
```
Two silent bugs in my GitLab CI pipeline. `changes:` was firing on tag pipelines where GitLab has no stable diff - so it matched everything. And three SCA jobs were all downloading the Trivy DB in parallel, racing each other. Neither was loud. Both were wasteful.
```

**Post 2** (274 chars):
```
Fixed both: scoped every `changes:` to explicit `if:` guards, and added a `cache:warm:trivy` job in `.pre` that downloads the DB once. SCA jobs flip to `policy: pull`. One download per pipeline. No race. Full writeup in CI/CD Lab 06. #CI #DevOps #GitLabCI #Trivy #HandsOnEngineer
```

---

## 💼 LinkedIn

```
Two bugs in my GitLab CI pipeline that made no noise but wasted time on every single run.

The first: `changes:` filters on cache jobs were firing on tag pipelines. GitLab has no stable diff reference on a tag pipeline, so it falls back to treating every file as changed. Jobs that should have been skipped were running every time I pushed a tag.

The second: three SCA jobs (vuln, license, sbom) all running in parallel and each independently downloading the Trivy vulnerability database. The first job had `policy: pull-push`, but the other two started at the same time so there was no guarantee they'd find a warm cache. In practice, all three downloaded from the internet on every run.

In CI/CD Lab 06, I cover how I fixed both:

- Scoped every bare `changes:` block to an explicit `if: $CI_PIPELINE_SOURCE` guard so tag pipelines are excluded cleanly
- Added a dedicated `cache:warm:trivy` job in `.pre` that runs `trivy --download-db-only` once and pushes to cache before any SCA job starts
- Flipped all three SCA jobs to `policy: pull` and added `needs: [cache:warm:trivy]` to enforce ordering
- Fixed `sca:sbom` image config to use `entrypoint: [""]` — same pattern the other Trivy jobs already had

One Trivy DB download per pipeline instead of three. No race condition. No more spurious cache rebuilds on tag pushes.

If you're running parallel security scans in GitLab CI, these patterns are worth knowing.

#GitLabCI #DevSecOps #Trivy #SCA #CICD #DevOps #PipelineOptimization #HandsOnEngineer
```

---

## ✉️ Email Subject Lines

1. **Curiosity**: Why was GitLab running my cache jobs on every tag push?
2. **How I**: How I stopped three SCA jobs from racing to download the same database
3. **Direct**: Fix: GitLab CI `changes:` on tag pipelines and Trivy DB parallel downloads
4. **Series**: CI/CD Lab #06: Fixing cache rules and pre-warming Trivy in GitLab CI

---

## ☁️ Bluesky

**Post 1** (298 chars):
```
Two quiet bugs in my GitLab CI pipeline: `changes:` firing on tag pipelines with no stable diff, and three SCA jobs all racing to download the Trivy vulnerability database in parallel. Neither raised an alarm. Both added waste to every single pipeline run.
```

**Post 2** (292 chars):
```
Fixed with a scoped `if:` guard on every bare `changes:` rule, and a new `cache:warm:trivy` job in `.pre` that downloads the DB once. SCA jobs pull only. One download, no race. CI/CD Lab 06 has the full breakdown with the exact YAML. #GitLabCI #Trivy #DevSecOps
```

---

## 🧵 Threads

**Post 1** (487 chars):
```
I pushed a git tag and noticed my GitLab CI cache jobs running when they had no reason to. Dug in and found two separate bugs: `changes:` filters have no stable diff on tag pipelines, so they match everything. And my three SCA jobs were all downloading the Trivy vulnerability database at the same time, in parallel, racing each other. Both bugs were silent. Both were wasteful. CI/CD Lab 06 is about finding and fixing them.
```

**Post 2** (452 chars):
```
What the fixes actually look like:

- Every bare `changes:` block now has an explicit `if: $CI_PIPELINE_SOURCE` guard — tag pipelines excluded cleanly
- New `cache:warm:trivy` job in `.pre` runs `trivy --download-db-only` once per pipeline and pushes to cache
- All three SCA jobs flip to `policy: pull` + `needs: [cache:warm:trivy]`
- `sca:sbom` gets the `entrypoint: [""]` fix the other Trivy jobs already had
```

**Post 3** (312 chars):
```
Result: one Trivy DB download per pipeline instead of three, no race condition, and no more spurious cache rebuilds on tag pushes. Full post with the YAML configs at the link — it's part of the ongoing CI/CD Lab series on GitLab CI security pipelines. #GitLabCI #Trivy #DevSecOps #DevOps
```

---

## 📘 Facebook

```
Sharing the latest from the CI/CD Lab series — and this one's about two bugs that made no noise but ran on every single pipeline.

The first: GitLab's `changes:` keyword has no stable diff on tag pipelines, so it silently matches everything. Jobs I thought were scoped were running on every tag push. The fix is just an `if:` guard, but you'd never know you needed it unless you pushed a tag and actually looked.

The second: three SCA jobs all downloading the Trivy vulnerability database in parallel, racing each other. One pre-warm job in `.pre` fixes it completely.

Full writeup with the YAML if you're working with GitLab CI security pipelines. Have you run into `changes:` behaving unexpectedly on tag pipelines before?
```

---

**Hashtag rationale**: `#GitLabCI` and `#Trivy` are the primary tools by name — specific and searchable. `#DevSecOps` fits the security scanning context. `#CI`, `#DevOps`, `#HandsOnEngineer` maintain series continuity across posts. Kept `#SCA` off Twitter/X to save characters; included on LinkedIn where length isn't a constraint.
