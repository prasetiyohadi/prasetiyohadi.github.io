# Social Posts — Kube GitOps Lab Part 8: Crossplane v2 Sequential Upgrades

---

## 𝕏 Twitter/X

**Post 1** (hook + takeaway):

```
Crossplane v2: you can't jump v2.0 → v2.2 directly.

CRD migrations are version-gated. Skip v2.1 and the v2.2 controller expects state that was never prepared. Helm applies without error. Reconciliation breaks later.

One hop at a time. #Kubernetes #Crossplane
```

_(~230 chars)_

**Post 2** (CTA):

```
Kube GitOps Lab Part 8: sequential Crossplane upgrades via ArgoCD.

One YAML file. Two commits. Two kubectl wait commands per hop. 10-15 min total.

Full walkthrough → [link] #GitOps #ArgoCD #DevOps #HandsOnEngineer
```

_(~215 chars)_

---

## 💼 LinkedIn

```
After landing Crossplane v2.0.0 in the previous lab, I assumed getting to v2.2.0 would be straightforward — bump the version tag, let ArgoCD sync, done.

It's not.

Crossplane ships CRD schema migrations with each minor release. The controller at version N handles migrations from N-1 to N. Jump from v2.0 to v2.2 directly and the v2.1 CRD migrations never run. Helm won't warn you. Reconciliation errors will.

In Part 8 of the Kubernetes GitOps Lab I walk through the sequential upgrade path: v2.0.0 → v2.1.0 → v2.2.0, each hop verified with ArgoCD before proceeding.

What's covered:

• Why minor version skipping breaks Crossplane silently — and why Helm doesn't catch it
• Branch isolation for Phase 3 (feat/phase-3-crossplane-v2-migration-phase-3) with three ArgoCD Applications updated to track it
• The kubectl wait --for=jsonpath pattern for verifying Application sync and health — more reliable than --for=condition for ArgoCD-specific status fields
• Final image verification confirming the running container matches the target version, not just the Helm revision
• The v2.2 pipelineinspector alpha feature and why I'm leaving it off for now

Each hop is identical: one YAML change, one commit, two kubectl wait commands. 10-15 minutes total, clean git history showing exactly what changed and when.

If you're running Crossplane v2 and need to get current, this is the pattern.

[link to full post]

#Kubernetes #GitOps #Crossplane #ArgoCD #DevOps #PlatformEngineering #InfrastructureAsCode
```

---

## ✉️ Email Subject Lines

1. **Curiosity**: `Why can't you skip Crossplane minor versions?`
2. **How I**: `How I upgraded Crossplane from v2.0 to v2.2 without breaking CRD migrations`
3. **Direct**: `Crossplane sequential upgrades: v2.0 → v2.1 → v2.2 via ArgoCD`
4. **Series**: `Kube GitOps Lab #8: Getting Crossplane to v2.2, one hop at a time`

---

## ☁️ Bluesky

**Post 1** (hook + concept):

```
Crossplane v2: can't skip minor versions.

CRD migrations are version-gated. v2.0 → v2.2 direct means v2.1 migrations never run. Helm applies clean. Reconciliation breaks later.

Sequential path: v2.0 → v2.1 → v2.2. kubectl wait between each hop. #Kubernetes #GitOps
```

_(~278 chars)_

**Post 2** (coverage + CTA):

```
Kube GitOps Lab Part 8 covers: branch isolation for Phase 3, --for=jsonpath as the reliable ArgoCD wait pattern (not --for=condition), image verification, and the v2.2 pipelineinspector alpha CRD.

Full post: [link] #Crossplane #ArgoCD
```

_(~239 chars)_

---

## 🧵 Threads

**Post 1**:

```
I just finished the sequential Crossplane upgrades: v2.0.0 → v2.1.0 → v2.2.0, all through ArgoCD, no helm upgrade commands run by hand.

The surprising part: you can't skip v2.1. Crossplane CRD migrations are version-gated. Helm won't stop you from trying. Your controllers will just silently break later. This is the part the official upgrade guide doesn't surface clearly.
```

**Post 2**:

```
Kube GitOps Lab Part 8 covers:

• Why minor version skipping breaks Crossplane — CRD migrations only run when the matching controller version is present
• The --for=jsonpath ArgoCD wait pattern (more reliable than --for=condition for Application status fields)
• Final image verification: confirming crossplane/crossplane:v2.2.0 is actually running, not just what Helm thinks it deployed
```

**Post 3**:

```
Each upgrade hop: one YAML change, one commit, two kubectl wait commands. 10-15 minutes total for both hops.

This is Part 8 of the Kubernetes GitOps Lab — the last step before wiring up a real cloud provider and making XCluster claims do something beyond writing a ConfigMap.

Full post: [link]
```

---

## 📘 Facebook

```
Just published Part 8 of the Kubernetes GitOps Lab: getting Crossplane from v2.0.0 to v2.2.0 via sequential minor version upgrades.

Turns out you can't skip v2.1 — Crossplane's CRD migrations are version-gated, and Helm won't warn you when you try. The upgrade itself is simple: one YAML file, two commits, two kubectl wait commands per hop. The important part is understanding why the sequence matters.

Includes the --for=jsonpath ArgoCD verification pattern I use for all Crossplane upgrades now.

Full post: [link]

Has anyone else hit silent reconciliation issues from skipping a Crossplane minor version? Curious what errors surfaced first.
```

---

_Hashtags chosen: `#Kubernetes #GitOps #Crossplane #ArgoCD` as primary topic tags — they match the exact tools covered. `#DevOps #PlatformEngineering` for reach on LinkedIn. `#HandsOnEngineer` as series identifier on X. `#InfrastructureAsCode` on LinkedIn for IaC-focused audience._
