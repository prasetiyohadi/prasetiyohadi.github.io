# Kube GitOps Lab 08: Crossplane v2 Sequential Upgrades: Getting to v2.2 the GitOps Way

---

In the [previous post](https://medium.com/), I did the hard part of the Crossplane v2 migration: jumping the core from v1.20.5 all the way to v2.0.0, installing `function-patch-and-transform` as v2's replacement for the removed native patch engine, and migrating the `XCluster` Composition from `spec.resources` to `spec.mode: Pipeline`. There were four gotchas worth documenting; the `Replace=True` requirement on the compositions ArgoCD Application took the most time to figure out.

Phase 2 left the cluster structurally ready for v2. But v2.0.0 is not where I want to stay. v2.2.0 is the latest stable release with bug fixes and improvements, and the path there requires two sequential hops, you can't skip minor versions with Crossplane.

In this post I do exactly that: bump from v2.0.0 to v2.1.0, verify, then bump from v2.1.0 to v2.2.0 and verify again. The whole thing runs in about 10-15 minutes and touches a single YAML file twice. This is the least dramatic part of the migration. It's also the cleanest demonstration of what GitOps buys you for routine upgrades.

The source code is on GitLab: [https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-3-crossplane-v2-migration-phase-3](https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-3-crossplane-v2-migration-phase-3)

---

### Why You Can't Skip Minor Versions

Before getting into the changes, it's worth understanding why the sequential path matters.

Crossplane ships CRD schema migrations with each minor release. The controller running at version N is responsible for upgrading CRD state from version N-1 to N. If you jump directly from v2.0 to v2.2, the v2.1 CRD migrations never run; the v2.2 controller comes up expecting state that hasn't been prepared. You may see reconciliation errors, missing fields, or controllers that silently skip managed resources because they don't match the expected schema version.

The Helm chart itself will apply without error because Helm doesn't validate CRD migration ordering. The problems show up later, during reconciliation. Sequential upgrades are the safer path, and with ArgoCD it's no more work than doing them all at once, you just commit twice instead of once.

---

### New Branch: `feat/phase-3-crossplane-v2-migration-phase-3`

Same branch isolation strategy as Phases 1 and 2. All Phase 3 changes live on `feat/phase-3-crossplane-v2-migration-phase-3`.

The three ArgoCD Applications tracking my GitLab repo get their `targetRevision` updated to point at the new branch:

```diff
   source:
     repoURL: https://gitlab.com/pras-labs/kube-gitops-demo.git
-    targetRevision: feat/phase-3-crossplane-v2-migration-phase-2
+    targetRevision: feat/phase-3-crossplane-v2-migration-phase-3
```

This applies to:

- **`apps/08-crossplane-providers.yaml`**: tracks `crossplane/providers/manifests/`
- **`apps/09-crossplane-providerconfigs.yaml`**: tracks `crossplane/providers/configs/`
- **`apps/10-crossplane-compositions.yaml`**: tracks `crossplane/compositions/`

The Crossplane core app (`apps/07-crossplane.yaml`) doesn't track a GitLab branch, it points directly at the Helm chart registry. That's the one getting the version bumps.

Rollback for all of Phase 3: revert the three `targetRevision` fields back to `phase-2`. ArgoCD restores the previous provider, config, and composition state automatically.

---

### Step 3a: Bumping v2.0.0 to v2.1.0

The change is one line in `apps/07-crossplane.yaml`:

```diff
 # Chart: https://charts.crossplane.io/stable
-# Version: 2.0.0
+# Version: 2.1.0

   source:
     repoURL: https://charts.crossplane.io/stable
     chart: crossplane
-    targetRevision: 2.0.0
+    targetRevision: 2.1.0
```

Commit and push:

```bash
git commit -m "chore(crossplane): upgrade core from v2.0.0 to v2.1.0" && git push
```

Then wait for ArgoCD to sync and report healthy:

```bash
kubectl wait --for=jsonpath='{.status.sync.status}'=Synced apps/crossplane -n argocd --timeout=180s
kubectl wait --for=jsonpath='{.status.health.status}'=Healthy apps/crossplane -n argocd --timeout=180s
```

I use `--for=jsonpath` rather than `--for=condition` here because ArgoCD Application health and sync status live in the status subresource fields, not Kubernetes condition types. The `jsonpath` form is more reliable for this specific resource.

Once both conditions pass, the v2.1.0 controller is running and the v2.1 CRD migrations have completed. That's the prerequisite for the next hop.

---

### Step 3b: Bumping v2.1.0 to v2.2.0

Same pattern, one more line change:

```diff
 # Chart: https://charts.crossplane.io/stable
-# Version: 2.1.0
+# Version: 2.2.0

   source:
     repoURL: https://charts.crossplane.io/stable
     chart: crossplane
-    targetRevision: 2.1.0
+    targetRevision: 2.2.0
```

Commit and push:

```bash
git commit -m "chore(crossplane): upgrade core from v2.1.0 to v2.2.0" && git push
```

Wait for sync and health again:

```bash
kubectl wait --for=jsonpath='{.status.sync.status}'=Synced apps/crossplane -n argocd --timeout=180s
kubectl wait --for=jsonpath='{.status.health.status}'=Healthy apps/crossplane -n argocd --timeout=180s
```

This is the same wait as Step 3a. I run it again rather than assuming it's fine, ArgoCD occasionally takes a moment longer on minor version bumps when provider pods restart and reconciliation queues rebuild.

---

### Final Verification

Once both hops complete, I verify the running image matches the target version:

```bash
kubectl get pods -n crossplane-system -o jsonpath="{.items[*].spec.containers[*].image}" | grep crossplane
# crossplane/crossplane:v2.2.0
```

If the image shows `v2.2.0`, the upgrade is confirmed end-to-end; not just the ArgoCD Application status, but the actual running container.

v2.2 also ships with an alpha feature called `pipelineinspector`, which adds a new CRD for debugging Pipeline-mode Compositions. It's not enabled by default and the CRD may not be present depending on your Helm values. I check for it as a sanity step, but absence is expected and OK:

```bash
kubectl get crds | grep pipelineinspector || echo "pipeline inspector CRD not present (alpha feature, OK)"
```

If you want to experiment with it, it requires enabling the `alpha` feature gate in your Crossplane Helm values. For this lab I'm leaving it off, the Pipeline-mode Composition is working and I don't need alpha APIs yet.

Rollback from any point in Phase 3 is the same as always: revert the version tag in the YAML, push, ArgoCD reconciles back. The Helm chart manages the CRD state on downgrade: going from v2.2 -> v2.1 -> v2.0 follows the same sequential logic in reverse.

---

### Conclusion

Phase 3 is the cleanest part of the entire Crossplane v2 migration:

- **Crossplane Core** bumped from v2.0.0 to v2.1.0, then v2.1.0 to v2.2.0; two sequential minor version hops, each verified before proceeding
- **Branch isolation** on `feat/phase-3-crossplane-v2-migration-phase-3`; the three ArgoCD Applications tracking GitLab manifests now follow this branch
- **ArgoCD `kubectl wait --for=jsonpath`** as the verification pattern for Application sync and health; more reliable than `--for=condition` for ArgoCD-specific status fields
- **Image verification** confirms the running controller matches the target version, not just the Helm chart revision
- **`pipelineinspector` CRD** is a v2.2 alpha feature; present if enabled in Helm values, absent by default, safe to ignore for now

The practical takeaway here is about the upgrade pattern itself. Each hop is identical: one YAML change, one commit, two `kubectl wait` commands. The whole upgrade to v2.2 takes 10-15 minutes and leaves a clear git history showing exactly what changed and when. If anything goes wrong at any step, the revert is a single commit; no Helm state to manually reconstruct, no out-of-band commands to undo.

That auditability is the compounding benefit of managing infrastructure through GitOps. By the time you reach Phase 3, the pattern is muscle memory.

---

### What's Next

The migration is done. Crossplane is on v2.2.0, the Composition is in Pipeline mode, and the branch structure is clean.

The original goal, the one I set up way back in Part 5 when I first introduced Crossplane for multi-cluster provisioning, was always to get to a real cloud provider and real `XCluster` claims that provision actual infrastructure. The migration work in Parts 6, 7, and 8 was the groundwork to get there on a stable v2 foundation.

That's what's coming: wiring up a real provider, giving Crossplane credentials that matter, and making `XCluster` do something more interesting than write a ConfigMap to the local cluster.

Stay tuned for next stories with Pras' Labs: The Hands-On Engineer!

---

_Tags: Kubernetes, GitOps, Crossplane, ArgoCD, DevOps, Platform Engineering, Infrastructure as Code, Helm_
