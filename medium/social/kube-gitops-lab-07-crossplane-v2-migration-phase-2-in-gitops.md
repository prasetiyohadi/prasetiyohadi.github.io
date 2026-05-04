# Social Posts — Kube GitOps Lab Part 7: Crossplane v2 Migration Phase 2

---

## 𝕏 Twitter/X

**Post 1** (hook):
```
Crossplane v2 removes native patch/transform. Your Compositions break on upgrade.

The fix: migrate to Pipeline mode + install function-patch-and-transform. But ArgoCD will also fail on the schema change unless you add Replace=True to syncOptions.

Took me two failed migrations to find that one.
```

**Post 2** (CTA):
```
Kubernetes GitOps Lab Part 7: Crossplane v2 migration Phase 2.

4 gotchas, the full Resources → Pipeline composition migration, and why managementPolicies: [Observe] is your best friend when live MRs are in the way.

Full post: [link]

#Kubernetes #Crossplane #GitOps #DevOps #HandsOnEngineer
```

---

## 💼 LinkedIn

```
Upgrading Crossplane to v2 looks straightforward on paper. One line change in a YAML file, ArgoCD syncs, done.

The reality: v2 removes the native patch and transform engine your Compositions depend on. If you have any Composition using spec.resources with inline patches, v2 will reject it — and the error message won't tell you that's why.

In Part 7 of my Kubernetes GitOps Lab series, I documented the full Phase 2 migration: v1.20.5 → v2.0.0, Composition mode migration from Resources to Pipeline, and the four gotchas I only found by running the migration twice.

What the post covers:
- Why ArgoCD needs Replace=True when you change a Composition schema (the error looks like a field conflict, not a schema incompatibility)
- How to install function-patch-and-transform — the v2 replacement for the built-in patch engine
- The full before/after YAML for migrating a Composition from spec.resources to spec.mode: Pipeline, with every structural difference explained
- How to protect live Managed Resources during migration using managementPolicies: [Observe]

All changes go through ArgoCD by editing YAML in Git. No helm upgrade commands, no manual kubectl apply. Rollback is a git revert.

If you're planning a Crossplane v2 upgrade, the gotchas section alone will save you a few hours.

Full post: [link]

#Kubernetes #GitOps #Crossplane #ArgoCD #DevOps #PlatformEngineering #InfrastructureAsCode
```

---

## ✉️ Email Subject Lines

1. `Why did my Crossplane upgrade break ArgoCD — and how I fixed it`
2. `How I migrated a Crossplane Composition from Resources to Pipeline mode (and what I got wrong twice)`
3. `Crossplane v2 migration: 4 gotchas, the Pipeline Composition migration, and ArgoCD Replace=True`
4. `Kubernetes GitOps Lab #7: Crossplane v2 Phase 2 — the major version jump`

---

## ☁️ Bluesky

**Post 1**:
```
Crossplane v2 drops native patch/transform.

Any Composition using spec.resources breaks on upgrade. You need to migrate to Pipeline mode + install function-patch-and-transform.

And ArgoCD will silently fail the schema change unless you add Replace=True.
```

**Post 2**:
```
Kubernetes GitOps Lab Part 7 covers the full migration: v1.20.5 → v2.0.0, before/after Composition YAML, protecting live MRs with managementPolicies: [Observe], and branch isolation for safe rollback.

Full post: [link]

#Kubernetes #Crossplane #GitOps
```

---

## 🧵 Threads

**Post 1**:
```
Crossplane v2 removes native patch and transform. That means every Composition using spec.resources breaks on upgrade.

I upgraded from v1.20.5 to v2.0.0 the GitOps way — all changes in YAML, all synced through ArgoCD — and documented the four things that surprised me along the way.

This is Part 7 of my Kubernetes GitOps Lab series.
```

**Post 2**:
```
What the post covers:

- Installing function-patch-and-transform — the v2 replacement for the removed patch engine
- Full Resources → Pipeline Composition migration (before/after YAML with every change explained)
- Why ArgoCD needs Replace=True for a Composition schema change (the error message is misleading)
- Protecting live Managed Resources using managementPolicies: [Observe]
```

**Post 3**:
```
Everything flows through ArgoCD. No helm upgrade commands, no manual state. The migration branch is wired into three ArgoCD app targetRevision fields — rollback is a git revert.

If you're planning a Crossplane v2 upgrade, the gotchas section is the part worth reading first.

Full post: [link]
```

---

## 📘 Facebook

```
Just published Part 7 of my Kubernetes GitOps Lab series — this one's about migrating Crossplane to v2 the GitOps way.

The short version: v2 removes the built-in patch engine, so every Composition needs to be rewritten for Pipeline mode. Getting it to sync cleanly in ArgoCD took two failed attempts before I found the Replace=True trick.

I documented everything — the before/after YAML, how to protect live resources during migration, and why the error message ArgoCD gives you doesn't actually tell you what's wrong.

If you've been putting off the v2 upgrade, have you run into any of these issues? Full post: [link]
```
