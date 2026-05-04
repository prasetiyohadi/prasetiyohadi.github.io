# Social Posts: Kube GitOps Lab 06 — Crossplane v2 Migration Phase 1

---

## 𝕏 Twitter/X

**Post 1** (271 chars):

```
Crossplane v2 is out. Wanted to add AWS providers. Turns out: can't upgrade v1.19 → v2 directly.

Required stepping stone: v1.19 → v1.20 → v2.0. Missed that in the docs.

Whole thing done via ArgoCD. No helm upgrade commands. Rollback = git revert.
```

**Post 2** (243 chars):

```
Wrote up the full Phase 1: pre-migration checklist, version bump, provider renaming (upbound-provider-helm → provider-helm), and GitOps branch isolation.

Kube GitOps Lab #6 👇

[your Medium link here]

#Kubernetes #GitOps #Crossplane #DevOps #HandsOnEngineer
```

---

## 💼 LinkedIn

```
I hit a wall trying to add AWS providers to my Crossplane setup.

Crossplane v2 has a hard sequential upgrade requirement. You can't jump from v1.19 directly to v2 — you have to land on v1.20 first, then proceed to v2.0, v2.1, v2.2. I didn't know that until I started reading the migration docs.

In Part 6 of my Kubernetes GitOps Lab series, I walk through Phase 1 of that migration — all of it done GitOps-native through ArgoCD, no manual helm commands:

• Pre-migration checklist: verifying no ControllerConfig resources, no native patch/transform Compositions, all provider packages using FQDNs
• Bumping Crossplane core from v1.19.0 to v1.20.5 — a single YAML change that ArgoCD syncs automatically
• Provider renaming: upbound-provider-helm and upbound-provider-kubernetes lose the upbound- prefix in v2
• GitOps branch isolation: the three downstream ArgoCD apps (providers, providerconfigs, compositions) all track a dedicated migration branch so rollback is a git revert

The GitOps approach is what makes this manageable. No Helm state to track manually. If something breaks at any step, git revert HEAD and ArgoCD reconciles back. That's the pattern I want to hold onto as the remaining phases get more complex.

Full write-up with all the YAML diffs and verification commands:
[your Medium link here]

#Kubernetes #GitOps #Crossplane #ArgoCD #DevOps #PlatformEngineering #InfrastructureAsCode
```

---

## ✉️ Email Subject Lines

1. **Curiosity**: `Why can't you upgrade Crossplane from v1.19 to v2 directly?`
2. **How I**: `How I migrated Crossplane to v2 without touching helm upgrade`
3. **Direct/practical**: `Crossplane v1 → v2 migration: checklist, YAML diffs, and rollback`
4. **Series continuity**: `Kube GitOps Lab #6: Crossplane v2 Migration — Phase 1`

---

## ☁️ Bluesky

**Post 1** (298 chars):

```
Wanted to add AWS providers to my Crossplane lab. Found out: can't jump v1.19 → v2 directly. Requires v1.20 as a stepping stone first.

Whole migration is GitOps-native — YAML changes in Git, ArgoCD syncs. Rollback = git revert. No manual helm state.

Kube GitOps Lab #6:
```

**Post 2** (287 chars):

```
Post covers: pre-migration checklist, bumping core to v1.20.5, provider renaming (upbound-provider-helm → provider-helm), and isolating migration on a dedicated branch so the 3 ArgoCD apps can roll back independently.

[your Medium link here]

#Kubernetes #GitOps #Crossplane
```

---

## 🧵 Threads

**Post 1** (488 chars):

```
I tried to add AWS providers to my Crossplane setup and hit a wall immediately.

Crossplane v2 is out. Newer community providers only publish v2-compatible packages. So I need to upgrade the core first.

Catch: you can't upgrade v1.19 → v2 directly. The upgrade path is v1.19 → v1.20 → v2.0 → v2.1 → v2.2. Sequential only.

This is Part 6 of my Kubernetes GitOps Lab — the unglamorous but necessary groundwork before real cloud provisioning.
```

**Post 2** (496 chars):

```
What Phase 1 covers:

• Pre-migration checklist — verify no ControllerConfigs, no native patch/transform Compositions, all packages use FQDNs
• Bump apps/07-crossplane.yaml: targetRevision 1.19.0 → 1.20.5 (one line, ArgoCD handles the rest)
• Provider rename: upbound-provider-helm → provider-helm, upbound-provider-kubernetes → provider-kubernetes (v2 drops the prefix)
• Branch isolation: 3 ArgoCD apps all track feat/phase-3-crossplane-v2-migration-phase-1
```

**Post 3** (312 chars):

```
The GitOps approach keeps rollback simple at every step: git revert HEAD, git push, ArgoCD reconciles back. No manual Helm state to manage.

Full write-up with YAML diffs and verification commands on Medium:

[your Medium link here]

#Kubernetes #GitOps #Crossplane #ArgoCD
```

---

## 📘 Facebook

```
Working through a Crossplane upgrade in my home lab this week and ran into something I hadn't expected: you can't jump directly from v1.19 to v2. There's a required stepping stone at v1.20 first, then sequential minor upgrades to v2.2.

Wrote up the full Phase 1 process — checklist, YAML changes, provider renaming, and how GitOps branch isolation keeps rollback simple at every step. Everything runs through ArgoCD, no manual helm commands.

Sharing in case anyone else is hitting this with their Crossplane setup. Would love to hear how others are handling the v2 migration.

[your Medium link here]
```

---

_Hashtags chosen: `#Kubernetes #GitOps #Crossplane #ArgoCD` — core tools in the post. `#DevOps #PlatformEngineering #InfrastructureAsCode` on LinkedIn where longer tag lists index well. `#HandsOnEngineer` on X to tie to series brand. Kept Bluesky/Threads tags minimal — character budgets are tight._
