# Kube GitOps Lab 07: Crossplane v2 Migration Phase 2: The Major Version Jump

---

In the [previous post](https://medium.com/), I walked through the first phase of the Crossplane v2 migration: bumping the core from v1.19.0 to v1.20.5 as the required stepping stone, and renaming providers from `upbound-provider-*` to `provider-*` to align with v2 conventions. All through ArgoCD, no `helm upgrade` commands run by hand.

Phase 1 was the unglamorous groundwork. Phase 2 is where things get more interesting, and where the gotchas live.

In this post I do the major version jump from v1.20.5 to v2.0.0, install the `function-patch-and-transform` Function that v2 requires for patch operations, and migrate the `XCluster` Composition from the v1 `spec.resources` syntax to `spec.mode: Pipeline`. I also document four gotchas I discovered through experimentation that would have cost me hours if I'd hit them cold.

The source code is on GitLab: [https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-3-crossplane-v2-migration-phase-2](https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-3-crossplane-v2-migration-phase-2)

---

### Gotchas Before You Proceed

Before getting into the actual changes, I want to document what I found through trial and error. These aren't in the official upgrade guide, they're the kind of things you find by reading error messages carefully and running the migration twice.

**Gotcha 1: Provider `matchLabels` naming matters more than it looks.**

In v1, provider deployment `matchLabels` are keyed differently than in v2. To maintain backward compatibility during the upgrade, always use the `provider-<name>` format in provider metadata, the rename Phase 1 already handled. If you kept any old `upbound-provider-*` names around, the v2 core may have trouble locating the correct provider pod during reconciliation.

**Gotcha 2: v2 removes native patch and transform.**

This is the breaking change. Crossplane v2 drops the built-in patch and transform engine that v1 Compositions used via `spec.resources`. Any Composition using that pattern will be rejected by v2. You need to migrate to `spec.mode: Pipeline` using an external Function. The Crossplane CLI (v1.20+) has a migration helper, but I converted manually, the schema change is understandable once you see it side by side.

**Gotcha 3: ArgoCD needs `Replace=True` for the Composition schema change.**

When you change a Composition from `spec.resources` to `spec.mode: Pipeline`, the structural difference is significant enough that ArgoCD's default server-side apply fails. The error isn't immediately obvious, it looks like a field conflict rather than a schema incompatibility. The fix is to add `Replace=True` to the `syncOptions` on the compositions ArgoCD Application. This tells ArgoCD to delete and recreate the resource instead of patching it.

**Gotcha 4: Protect managed resources during migration.**

If any Managed Resources (MRs) were created by the Composition before you migrate it, the structural change can trigger unexpected reconciliation or deletion. The safest approach is to set `spec.deletionPolicy: Orphan` in the Composition before changing its mode, Crossplane will orphan existing MRs instead of deleting them if the Composition changes incompatibly.

Alternatively, patch the MRs directly to put them in observe-only mode:

```bash
kubectl patch object.kubernetes.crossplane.io/demo-cluster-4xwwt-vlwn7 \
  -p '{"spec":{"managementPolicies":["Observe"]}}' \
  --type=merge
```

Setting `managementPolicies: [Observe]` makes Crossplane watch the resource without reconciling or deleting it. It's a useful escape hatch any time you need to make structural composition changes on a cluster with live resources.

---

### Bumping Core to v2.0.0

With the gotchas documented, the actual version bump is the same one-file pattern as Phase 1. `apps/07-crossplane.yaml` gets a single line change:

```diff
 # Chart: https://charts.crossplane.io/stable
-# Version: 1.20.5
+# Version: 2.0.0

   source:
     repoURL: https://charts.crossplane.io/stable
     chart: crossplane
-    targetRevision: 1.20.5
+    targetRevision: 2.0.0
```

Commit, push, ArgoCD syncs. For verification I switched from `--for=condition=Healthy` to `--for=jsonpath`, ArgoCD Application status lives in the status subresource and `jsonpath` is more reliable for this:

```bash
kubectl wait --for=jsonpath='{.status.sync.status}'=Synced apps/crossplane -n argocd --timeout=180s
kubectl wait --for=jsonpath='{.status.health.status}'=Healthy apps/crossplane -n argocd --timeout=180s

kubectl get deployment crossplane -n crossplane-system -o yaml | grep image:
# crossplane/crossplane:v2.0.0

kubectl get pods -n crossplane-system -o jsonpath="{.items[*].spec.containers[*].image}"
# Should include crossplane/crossplane:v2.0.0
```

Rollback is the same `git revert` approach from Phase 1. ArgoCD reconciles back to v1.20.5 without any manual Helm state to manage.

---

### Adding `function-patch-and-transform`

Since v2 removes the built-in patch engine, I need to install the official replacement: the `function-patch-and-transform` Crossplane Function published by Upbound. This is what Compositions will call via `functionRef` to do patch and transform operations.

I created `crossplane/providers/manifests/function-patch-and-transform.yaml`:

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-patch-and-transform
spec:
  package: xpkg.upbound.io/upbound/function-patch-and-transform:v0.10.3
```

A few things worth noting here:

- **`apiVersion: pkg.crossplane.io/v1beta1`**; Functions use `v1beta1`, not `v1` like Providers. They're a newer API introduced alongside the Pipeline Composition model.
- **`kind: Function`**; A distinct resource type from Provider. Crossplane installs it the same way (pulling the package from the registry) but it runs as a function runtime that Compositions invoke during reconciliation, rather than a controller watching external resources.
- **Package pinned to `v0.10.3`**; I verified this is the latest stable tag compatible with Crossplane v2.0.0 before pinning.

This file lives in `crossplane/providers/manifests/`, so the providers ArgoCD app picks it up automatically. Once the package is pulled and started, `kubectl get functions` will show `HEALTHY: True`.

---

### Migrating the Composition to Pipeline Mode

This is the main work of Phase 2. The `XCluster` Composition was written in v1 style: `spec.resources` with inline patches. v2 rejects this. It needs to become a Pipeline-mode Composition that delegates patching to the `function-patch-and-transform` Function.

Here's what the Composition looked like before:

```yaml
spec:
  compositeTypeRef:
    apiVersion: infrastructure.lab.local/v1alpha1
    kind: XCluster
  resources:
    - name: cluster-info
      base:
        apiVersion: kubernetes.crossplane.io/v1alpha2
        kind: Object
        spec:
          forProvider:
            manifest:
              apiVersion: v1
              kind: ConfigMap
              metadata:
                namespace: crossplane-system
              data:
                status: "provisioned"
                managed-by: "crossplane"
          providerConfigRef:
            name: default
      patches:
        - fromFieldPath: metadata.name
          toFieldPath: spec.forProvider.manifest.metadata.name
          transforms:
            - type: string
              string:
                fmt: "cluster-info-%s"
        - fromFieldPath: spec.parameters.environment
          toFieldPath: spec.forProvider.manifest.data.environment
        - fromFieldPath: spec.parameters.version
          toFieldPath: spec.forProvider.manifest.data.kubernetes-version
```

And after the migration:

```yaml
spec:
  compositeTypeRef:
    apiVersion: infrastructure.lab.local/v1alpha1
    kind: XCluster
  mode: Pipeline
  pipeline:
    - functionRef:
        name: function-patch-and-transform
      step: patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
          - name: cluster-info
            base:
              apiVersion: kubernetes.crossplane.io/v1alpha2
              kind: Object
              spec:
                forProvider:
                  manifest:
                    apiVersion: v1
                    kind: ConfigMap
                    metadata:
                      namespace: crossplane-system
                    data:
                      status: provisioned
                      managed-by: crossplane
                providerConfigRef:
                  name: default
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: metadata.name
                toFieldPath: spec.forProvider.manifest.metadata.name
                transforms:
                  - type: string
                    string:
                      type: Format
                      fmt: cluster-info-%s
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.environment
                toFieldPath: spec.forProvider.manifest.data.environment
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.version
                toFieldPath: spec.forProvider.manifest.data.kubernetes-version
```

What actually changed:

- **`spec.resources` is gone**; The top-level resources list is removed. The Composition now declares `mode: Pipeline` at the spec level.
- **`spec.pipeline`**; A list of steps. Each step names a Function via `functionRef` and provides configuration as `input`.
- **`functionRef.name: function-patch-and-transform`**; References the Function resource created in the previous step. Crossplane calls this Function during each reconciliation.
- **`input.apiVersion: pt.fn.crossplane.io/v1beta1`**; This is the Function's own API group, not Crossplane's. The `input` block is the Function's configuration schema, which the Function itself defines and validates.
- **`input.kind: Resources`**; The `function-patch-and-transform` Function expects its input as a `Resources` document. The resource definitions, base manifests, and patches all move inside this input block.
- **Explicit `type:` on patches**; In v1, `fromFieldPath`/`toFieldPath` patches were implicitly typed as `FromCompositeFieldPath`. In Pipeline mode, `type: FromCompositeFieldPath` is required on each patch. Same for transforms: `type: Format` is now explicit inside the string transform rather than implied.

The resource logic is identical: same `Object` base, same ConfigMap, same three patches. The structure just moved from `spec.resources[].patches` into `spec.pipeline[].input.resources[].patches`.

Because this is a structural schema change on an existing Composition, you'll also need to add `Replace=True` to `apps/10-crossplane-compositions.yaml` before syncing:

```yaml
syncPolicy:
  syncOptions:
    - Replace=True
```

Without this, ArgoCD will try to server-side apply the new Composition over the existing one and hit a field conflict error. `Replace=True` handles the delete-and-recreate cleanly.

---

### Branch Strategy Update

Same isolation pattern as Phase 1. All Phase 2 changes live on `feat/phase-3-crossplane-v2-migration-phase-2`. The three ArgoCD Applications tracking my repo get their `targetRevision` updated:

```diff
   source:
     repoURL: https://gitlab.com/pras-labs/kube-gitops-demo.git
-    targetRevision: feat/phase-3-crossplane-v2-migration-phase-1
+    targetRevision: feat/phase-3-crossplane-v2-migration-phase-2
```

This applies to `apps/08-crossplane-providers.yaml`, `apps/09-crossplane-providerconfigs.yaml`, and `apps/10-crossplane-compositions.yaml`. The new `function-patch-and-transform.yaml` manifest in `crossplane/providers/manifests/` is picked up by the providers app; the migrated Composition in `crossplane/compositions/` is picked up by the compositions app.

Rolling back Phase 2 means reverting these three `targetRevision` fields to point back to `phase-1`. ArgoCD restores the old provider manifests and the Resources-mode Composition automatically.

---

### Conclusion

Phase 2 is where the structural migration actually happens:

- **Crossplane Core** bumped from v1.20.5 to v2.0.0; the major version jump that Phase 1 set up as a stepping stone
- **`function-patch-and-transform`** installed as a new `Function` resource; the official Upbound replacement for v2's removed native patch engine
- **`XCluster` Composition** migrated from `spec.resources` to `spec.mode: Pipeline`; same logic, structure moved inside the Function's `input` block
- **ArgoCD `Replace=True`** required on the compositions app; handles the schema incompatibility during the migration sync
- **Branch isolation** continued on `feat/phase-3-crossplane-v2-migration-phase-2`; rollback to Phase 1 is a `targetRevision` revert

The gotchas section is the part I'd have wanted going in. The `Replace=True` requirement took the most time to figure out; the error message points at a field conflict, not at the Composition schema change being the root cause. And `managementPolicies: [Observe]` is a trick worth keeping in your back pocket beyond this migration: any time you're making structural composition changes on a cluster with live resources, it buys you a safe window to verify things before re-enabling full reconciliation.

---

### What's Next

With the core on v2.0.0 and the Composition migrated to Pipeline mode, the structural upgrade work is in place. The next steps are sequential minor upgrades through v2.1 and v2.2 to get current, and then back to the original plan: a real cloud provider, real cluster provisioning, and `XCluster` claims that do something more interesting than write a ConfigMap.

Stay tuned for next stories with Pras' Labs: The Hands-On Engineer!

---

_Tags: Kubernetes, GitOps, Crossplane, ArgoCD, DevOps, Platform Engineering, Infrastructure as Code, Helm_
