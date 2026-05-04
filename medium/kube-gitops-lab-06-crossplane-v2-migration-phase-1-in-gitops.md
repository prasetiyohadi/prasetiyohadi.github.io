# Kube GitOps Lab 06: Migrating Crossplane to v2 the GitOps Way: Starting with the Stepping-Stone

In the [previous post](https://medium.com/), I wired Crossplane into the GitOps setup: core installed via Helm in Wave 7, Helm and Kubernetes providers in Wave 8, ProviderConfigs in Wave 9, and a placeholder XRD with a ConfigMap Composition in Wave 10. I ended that post saying the next step was replacing the ConfigMap with real cloud resources: an AWS provider, EKS cluster provisioning, the works.

That's still the destination. But before I can get there, I need to deal with something more immediate: Crossplane v2 is out, and the upgrade path has a catch I didn't anticipate.

In this post, I walk through the first phase of the Crossplane v1 -> v2 migration in a GitOps-native way. That means no `helm upgrade` commands run by hand, everything goes through ArgoCD by changing YAML in Git. I'll cover why v2 requires an intermediate stop at v1.20, what changes in the provider naming convention, and how I isolated this migration work in its own branch so rollback stays a single `git revert`.

The source code is on GitLab: [pras-labs/kube-gitops-demo](https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-3-crossplane-v2-migration-phase-1)

---

### Why v2 Before Cloud Providers

When Crossplane v2 dropped, I took a look at what actually changed. The short version: v2 is backward compatible with existing XRDs and Compositions, but it introduces a new naming convention for community providers, renames some internal APIs, and drops support for a few v1 patterns (`ControllerConfig`, native patch/transform Compositions without `.mode: Pipeline`).

More importantly, several newer community providers are only publishing v2-compatible packages. If I want to use a modern AWS or GCP provider in the next phase, I need the core running on v2 first. Trying to do both upgrades at once, core and AWS provider, is asking for a bad time when something breaks.

> So: v2 migration now, cloud providers next.

---

### The Migration Roadmap

Crossplane has a strict sequential upgrade policy. You can't jump from v1.19 directly to v2. The upgrade path is:

```
v1.19.0 -> v1.20.x  ->  v2.0.0  ->  v2.1.x  ->  v2.2.x
   (current)  (stepping stone)  (major jump)  (sequential minors)
```

I mapped this out as a 5-phase plan, which I committed to `docs/phase-3.md` in the repo:

1. **Preparation**: Verify v1.19 state, upgrade to v1.20 (required stepping stone)
2. **Major jump**: v1.20 -> v2.0 (backward-compatible core upgrade)
3. **Catch up**: v2.0 -> v2.1 -> v2.2 (sequential minors)
4. **Adopt v2**: Update XRD API to v2, migrate Compositions to Pipeline mode
5. **Harden**: Configure MRAP, clean up legacy resources

This post covers Phase 1. Each phase gets its own commit and ArgoCD sync so I can verify health before moving forward.

---

### Pre-Migration Checklist

Before touching anything, I verified the current cluster state. The key concerns for a Crossplane v1 -> v2 migration are: compositions using the old native patch/transform mode (removed in v2), `ControllerConfig` resources (deprecated in v1.x, gone in v2), and provider package references that don't use fully qualified domain names.

```bash
# Check current versions
kubectl get pods -n crossplane-system -o wide | grep crossplane
kubectl get providers -A | grep crossplane
kubectl get packages -A | grep crossplane

# Verify no removed features in use
kubectl get compositions -A -o yaml | grep -i "mode: Resources" && echo "ERROR: Native patch/transform still in use"
kubectl get controllerconfigs -A && echo "WARNING: ControllerConfig resources found"
kubectl get packages -A | grep -v "xpkg\.crossplane\.io\|xpkg\.upbound\.io" && echo "ERROR: Non-FQDN packages found"

# Check ArgoCD health
kubectl get apps -n argocd | grep crossplane
kubectl get applications -n argocd crossplane crossplane-providers crossplane-providerconfigs crossplane-compositions -o yaml

# Verify no active Composite Resources or Managed Resources
kubectl get composite -A
kubectl get managed -A
```

In my lab, everything came back clean: no `ControllerConfig` resources, no native patch/transform Compositions (I only have the Pipeline-mode placeholder from Part 5), and all provider packages already use FQDN package references. Green light to proceed.

I also created a backup branch before starting:

```bash
git branch backup/crossplane-v1-$(date +%Y%m%d)
git push origin backup/crossplane-v1-$(date +%Y%m%d)
```

---

### Phase 1: Bumping the Core to v1.20.5

The v1.20 upgrade is a single file change. `apps/07-crossplane.yaml` is the ArgoCD Application that deploys the Crossplane Helm chart from `charts.crossplane.io/stable`. I bumped `targetRevision` from `1.19.0` to `1.20.5`:

```diff
 # Chart: https://charts.crossplane.io/stable
-# Version: 1.19.0
+# Version: 1.20.5

 apiVersion: argoproj.io/v1alpha1
 kind: Application
 ...
   source:
     repoURL: https://charts.crossplane.io/stable
     chart: crossplane
-    targetRevision: 1.19.0
+    targetRevision: 1.20.5
```

Commit, push, ArgoCD syncs automatically. I waited for the sync to complete and verified:

```bash
# Wait for ArgoCD to mark the app healthy
kubectl wait --for=condition=Healthy apps/crossplane -n argocd --timeout=180s

# Confirm the new image is running
kubectl get pods -n crossplane-system -o wide
kubectl get deployments -n crossplane-system crossplane -o yaml | grep image
# image: crossplane/crossplane:v1.20.x

# Confirm providers still show as Ready
kubectl get providers -A
```

The rollout was smooth; Crossplane handles its own CRD upgrades during the Helm upgrade, and v1.20 is fully backward compatible with my existing providers, ProviderConfigs, XRDs, and Compositions. Nothing broke.

If it had broken, rollback is a single command:

```bash
git revert HEAD
git push
# ArgoCD reverts to v1.19.0 automatically
kubectl wait --for=condition=Healthy apps/crossplane -n argocd --timeout=180s
```

That's the GitOps advantage here. No `helm rollback` commands, no manual state management; the Git history is the truth, and ArgoCD reconciles to it.

---

### Provider Renaming: Dropping the Upbound Prefix

Alongside the core upgrade, I cleaned up the provider names. In v1.x, the community Helm and Kubernetes providers were published under the `upbound-provider-*` naming convention. Starting with Crossplane v2, the community providers drop the `upbound-` prefix and go by `provider-helm` and `provider-kubernetes`.

This is purely a naming change in the Kubernetes resource metadata, the package references (`xpkg.upbound.io/crossplane-contrib/...`) stay the same. But the name matters because other resources reference these providers by name, and keeping old names while running v2 creates confusion when reading `kubectl get providers`.

`crossplane/providers/manifests/helm-provider.yaml`:

```diff
 apiVersion: pkg.crossplane.io/v1
 kind: Provider
 metadata:
-  name: upbound-provider-helm
+  name: provider-helm
 spec:
   package: xpkg.upbound.io/crossplane-contrib/provider-helm:v0.19.0
   runtimeConfigRef:
     name: provider-runtimeconfig
```

`crossplane/providers/manifests/kubernetes-provider.yaml`:

```diff
 apiVersion: pkg.crossplane.io/v1
 kind: Provider
 metadata:
-  name: upbound-provider-kubernetes
+  name: provider-kubernetes
 spec:
   package: xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.15.0
   runtimeConfigRef:
     name: provider-runtimeconfig
```

Kubernetes handles this as a delete + recreate, the old Provider resource gets removed and the new one is created with the updated name. Crossplane re-downloads the provider package and restarts the controller pod. It takes a minute or two for the new-named provider to show `HEALTHY: True`, but there's no data loss since these providers aren't managing any external resources yet.

---

### GitOps Branch Isolation

All the migration work happens on a dedicated branch: `feat/phase-3-crossplane-v2-migration-phase-1`. The three ArgoCD Applications that point to my repo for Crossplane configuration: providers; providerconfigs; and compositions, all had their `targetRevision` updated to track this branch:

`apps/08-crossplane-providers.yaml`:

```diff
   source:
     repoURL: https://gitlab.com/pras-labs/kube-gitops-demo.git
-    targetRevision: feat/phase-3-initial-crossplane-multi-cluster-provisioning
+    targetRevision: feat/phase-3-crossplane-v2-migration-phase-1
     path: crossplane/providers/manifests/
```

Same change in `apps/09-crossplane-providerconfigs.yaml` (path: `crossplane/providers/configs/`) and `apps/10-crossplane-compositions.yaml` (path: `crossplane/compositions/`).

This branch isolation matters for two reasons. First, it lets me iterate on provider and composition manifests without touching main. Second, rolling back any of these ArgoCD apps means just reverting the `targetRevision` change, ArgoCD syncs back to the previous branch and the old manifests are restored.

In a team setting, this is also where you'd gate on a review before merging to main. The ArgoCD apps can trail behind main on a feature branch for as long as the migration is in-flight.

---

### Conclusion

Phase 1 of the Crossplane v2 migration is the unglamorous but necessary groundwork:

- **Crossplane Core** bumped from v1.19.0 to v1.20.5; the required stepping stone before the v2 major jump
- **Provider names** cleaned up from `upbound-provider-helm`/`upbound-provider-kubernetes` to `provider-helm`/`provider-kubernetes`; aligning with v2 naming conventions
- **Migration branch** created and wired into ArgoCD app `targetRevision` fields; all three downstream apps (providers, providerconfigs, compositions) now track `feat/phase-3-crossplane-v2-migration-phase-1`
- **Rollback** available at every step via `git revert`; no manual Helm state to manage

The pre-migration checklist is worth keeping. Running it before any future Crossplane upgrade takes two minutes and surfaces problems before you're mid-upgrade wondering why things look wrong.

---

### What's Next

Phase 1 put me on v1.20.5. Phase 2 is the major jump: upgrading from v1.20 to v2.0, then sequentially through v2.1 and v2.2. That's where the more interesting changes land: new CRD APIs, the shift to Pipeline-mode Compositions as the standard, and the eventual adoption of the v2 XRD API.

After the core is on v2.2, Phase 3 is back to the original plan: a real cloud provider, real cluster provisioning, real `XCluster` claims that don't resolve to ConfigMaps.

Stay tuned for next stories with Pras' Labs: The Hands-On Engineer!

---

_Tags: Kubernetes, GitOps, Crossplane, ArgoCD, DevOps, Platform Engineering, Multi-Cluster, Infrastructure as Code, Helm_
