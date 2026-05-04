# Kube GitOps Lab 09: Crossplane v2 Phase 4: Upgrading Providers and Adopting Namespace-Scoped Objects

---

In the [previous post](https://medium.com/), I completed the sequential minor version upgrades for Crossplane core: v2.0.0 to v2.1.0, then v2.1.0 to v2.2.0. Each hop was one YAML change, one commit, two `kubectl wait` commands. By the end, the Crossplane runtime was fully on v2: the core controller, the Pipeline-mode Composition, the `function-patch-and-transform` function.

What I hadn't updated yet: the providers themselves. `provider-helm` was still at `v0.19.0` from the `crossplane-contrib` org. `provider-kubernetes` was at `v0.15.0`, same org. Both are now superseded by Upbound-published versions with major version bumps and updated API groups. Phase 4 is about closing that gap: upgrading the providers to their v1.x releases, switching to the official Upbound registry namespace, and updating the Composition to use the namespace-scoped Object API that the new kubernetes provider ships.

There were two gotchas worth documenting: a ProviderRevision conflict that blocked the new providers from becoming healthy, and an XRD scope field I tried to change that turned out to be immutable. Both required manual intervention that the GitOps loop couldn't handle on its own.

The source code is on GitLab: [https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-3-crossplane-v2-migration-phase-4](https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-3-crossplane-v2-migration-phase-4)

---

### New Branch: `feat/phase-3-crossplane-v2-migration-phase-4`

Same branch isolation pattern as previous phases. All Phase 4 changes live on `feat/phase-3-crossplane-v2-migration-phase-4`.

The three ArgoCD Applications tracking my GitLab repo get their `targetRevision` updated:

```diff
   source:
     repoURL: https://gitlab.com/pras-labs/kube-gitops-demo.git
-    targetRevision: feat/phase-3-crossplane-v2-migration-phase-3
+    targetRevision: feat/phase-3-crossplane-v2-migration-phase-4
```

This applies to:

- **`apps/08-crossplane-providers.yaml`**: tracks `crossplane/providers/manifests/`
- **`apps/09-crossplane-providerconfigs.yaml`**: tracks `crossplane/providers/configs/`
- **`apps/10-crossplane-compositions.yaml`**: tracks `crossplane/compositions/`

Rollback for all of Phase 4: revert those three `targetRevision` fields back to `phase-3`. ArgoCD reconciles the previous provider versions back.

---

### Upgrading Providers: From `crossplane-contrib` to `upbound`

The providers were originally installed from the `crossplane-contrib` GitHub org, the community-maintained collection. Upbound has since taken over official publishing of the core providers under their own registry namespace. For `provider-helm` and `provider-kubernetes`, that means switching from `crossplane-contrib` to `upbound` in the package reference and picking up the v1.x releases.

`crossplane/providers/manifests/helm-provider.yaml`:

```diff
 spec:
-  package: xpkg.upbound.io/crossplane-contrib/provider-helm:v0.19.0
+  # Source: https://marketplace.upbound.io/providers/crossplane-contrib/provider-helm/v1.2.1
+  package: xpkg.upbound.io/upbound/provider-helm:v1.2.1
   runtimeConfigRef:
     name: default
```

`crossplane/providers/manifests/kubernetes-provider.yaml`:

```diff
 spec:
-  package: xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.15.0
+  # Source: https://marketplace.upbound.io/providers/upbound/provider-kubernetes/v1.2.2
+  package: xpkg.upbound.io/upbound/provider-kubernetes:v1.2.2
   runtimeConfigRef:
     name: default
```

The version jump is significant: `v0.19` to `v1.2` for helm, `v0.15` to `v1.2` for kubernetes. These aren't patch updates; both packages crossed a major version boundary and ship new API groups alongside the old ones.

#### Gotcha: ProviderRevision Conflicts

After committing the provider updates and letting ArgoCD sync, I ran into a situation where the new provider pods came up but the Providers stayed in a non-healthy state. The symptom: `kubectl get providers` showed the new revision being installed but never becoming `HEALTHY=True`.

The root cause is documented in [crossplane/crossplane#5801](https://github.com/crossplane/crossplane/discussions/5801): when you switch the package reference (especially across registry orgs), Crossplane creates a new ProviderRevision but the old one is still present. The old revision's controller loop conflicts with the new one during initialization. The GitOps reconciliation loop doesn't resolve this on its own.

The fix is manual: delete the stale ProviderRevision objects and let Crossplane recreate clean ones.

```bash
# List all ProviderRevisions
kubectl get providerrevisions

# Delete the old revision(s) for each provider, the one not matching the new package version
kubectl delete providerrevision <old-revision-name>
```

After deleting the stale revisions, the new providers came up healthy within a minute. ArgoCD's health check confirmed:

```bash
kubectl wait --for=jsonpath='{.status.conditions[?(@.type=="Healthy")].status}'=True \
  providers/provider-helm --timeout=120s

kubectl wait --for=jsonpath='{.status.conditions[?(@.type=="Healthy")].status}'=True \
  providers/provider-kubernetes --timeout=120s
```

This is one of those cases where GitOps manages desired state but can't handle cleanup of orphaned objects. The stale ProviderRevision isn't in Git, it was created by Crossplane itself, so ArgoCD can't prune it. Manual intervention is the right call here, not a workaround.

---

### XRD Scope: `LegacyCluster`

With the new kubernetes provider installed, I wanted to understand how the scope model changed in v2. The XRD for `XCluster` had no explicit `spec.scope` field; Crossplane inferred cluster scope by default.

In Crossplane v2, `spec.scope` on a CompositeResourceDefinition is explicit. The options are:

- **`LegacyCluster`**: cluster-scoped XRs that also support Claims; this is the v1 behavior, carried forward
- **`Namespaced`**: namespace-scoped XRs (no Claims support in v2)

I first tried setting `spec.scope: Namespaced` to see what would happen. ArgoCD reported a sync error immediately:

```
CompositeResourceDefinition.apiextensions.crossplane.io "xclusters.infrastructure.lab.local" is invalid:
[spec: Invalid value: "object": Only LegacyCluster composite resources can offer claims,
spec.scope: Invalid value: "string": Value is immutable]
```

Two problems in one error:

1. The XRD already had `claimNames` defined, Claims are only valid for `LegacyCluster` scope
2. `spec.scope` is immutable, once an XRD is created, you can't change it

The fix is straightforward: set `spec.scope: LegacyCluster` explicitly, which matches the existing behavior. This satisfies Crossplane v2's requirement for an explicit scope declaration without changing anything functionally.

```diff
 spec:
+  scope: LegacyCluster
   group: infrastructure.lab.local
   names:
     kind: XCluster
```

The immutability constraint means if you wanted to switch a live XRD from `LegacyCluster` to `Namespaced`, you'd need to delete the XRD and all associated XRs first, a disruptive migration that requires careful planning and active workload consideration. For this lab, keeping `LegacyCluster` is the right call.

---

### Composition Object API: `kubernetes.m.crossplane.io/v1alpha1`

The `provider-kubernetes` v1.x release ships a new API group alongside the original one:

- **`kubernetes.crossplane.io`**: original group, cluster-scoped Objects
- **`kubernetes.m.crossplane.io`**: new group, namespace-scoped Objects (the `m` prefix stands for "managed namespace")

The Composition was referencing the old `v1alpha2` Object from the original group. Updating to the new provider means switching to `v1alpha1` from the `m`-prefixed group:

```diff
         resources:
           - base:
-              apiVersion: kubernetes.crossplane.io/v1alpha2
+              # Namespace scoped
+              apiVersion: kubernetes.m.crossplane.io/v1alpha1
               kind: Object
               spec:
                 forProvider:
```

The practical difference: Objects created via `kubernetes.m.crossplane.io/v1alpha1` live in a namespace rather than being cluster-scoped. For a lab environment that provisions workload clusters, namespace-scoped Objects are a cleaner boundary; the managed Objects belong to the namespace of the claim or XR, not floating at cluster scope.

The `kind: Object` stays the same. Only the API group and version change.

---

### Decision: Keep v1 XRD or Go Full v2?

At this point the question came up: should I migrate the XRD itself to `apiextensions.crossplane.io/v2`?

Crossplane v2 ships a new XRD API (`apiextensions.crossplane.io/v2`) that differs from v1 in one significant way: v2 XRDs don't support `claimNames`. Claims are a v1 concept. The v2 design treats namespace-scoped XRs (via `spec.scope: Namespaced`) as the replacement.

The current XRD uses `apiextensions.crossplane.io/v1` with `claimNames` defined. It's fully supported by the Crossplane v2 runtime; v1 XRDs work fine, the runtime is backwards-compatible. Migrating to v2 XRD would require:

- Removing `claimNames` (no claim support)
- Switching to `apiextensions.crossplane.io/v2`
- Updating any Composition logic that relies on claim-derived names

For this lab, I'm keeping v1 XRD. The `claimNames` pattern is useful for the eventual provisioning demo, and switching to v2 XRD now would change the abstraction before I've used it for anything real. The v1 XRD is already v2-runtime-compatible; there's nothing to gain from forcing the API version upgrade at this stage.

The ProviderConfigs also got explicit scope comments for clarity:

```diff
+# Cluster scoped
 apiVersion: helm.crossplane.io/v1beta1
 kind: ProviderConfig
```

```diff
+# Cluster scoped
 apiVersion: kubernetes.crossplane.io/v1alpha1
 kind: ProviderConfig
```

These are documentation-only changes. ProviderConfigs in Crossplane are cluster-scoped by design, the comment makes that explicit for anyone reading the manifests.

---

### Conclusion

Phase 4 closes the gap between the Crossplane v2 core migration and the provider ecosystem:

- **`provider-helm`** upgraded from `crossplane-contrib/provider-helm:v0.19.0` to `upbound/provider-helm:v1.2.1`: official Upbound registry, major version release
- **`provider-kubernetes`** upgraded from `crossplane-contrib/provider-kubernetes:v0.15.0` to `upbound/provider-kubernetes:v1.2.2`: same migration, ships the new `kubernetes.m.crossplane.io` namespace-scoped API group
- **ProviderRevision cleanup** required manually: old revisions conflict with new ones during package re-registration, and ArgoCD can't prune objects Crossplane creates outside of Git
- **XRD `spec.scope: LegacyCluster`** set explicitly: v2 requires the field, it's immutable once set, attempting `Namespaced` on an XRD with `claimNames` fails immediately
- **Composition Object API** switched from `kubernetes.crossplane.io/v1alpha2` to `kubernetes.m.crossplane.io/v1alpha1`: namespace-scoped Objects from the updated provider
- **v1 XRD kept intentionally**: v1 XRDs with `claimNames` are fully compatible with the v2 runtime; migrating to v2 XRD would require dropping claim support, which changes the lab's provisioning pattern

The pattern across all four phases of this migration has been consistent: GitOps handles desired state, but not cleanup of objects that Crossplane manages outside of Git. ProviderRevisions, old CRD fields, orphaned resources; those require targeted manual intervention. The trick is knowing which operations fall inside the GitOps loop and which ones don't.

---

### What's Next

The migration is complete. Crossplane is on v2.2.0, providers are on their latest official releases, the Composition uses v2-era APIs, and the XRD scope is explicit.

The original goal when I introduced Crossplane back in Part 5 was multi-cluster provisioning: actual `XCluster` claims that spin up real infrastructure, not just a ConfigMap on the local cluster. The migration work in Parts 6 through 9 was the prerequisite: getting onto a stable v2 foundation before wiring up real cloud credentials.

That's what's coming next: a real cloud provider, actual credentials, and an `XCluster` claim that provisions something you can SSH into.

Stay tuned for next stories with Pras' Labs: The Hands-On Engineer!

---

_Tags: Kubernetes, GitOps, Crossplane, ArgoCD, DevOps, Platform Engineering, Infrastructure as Code_
