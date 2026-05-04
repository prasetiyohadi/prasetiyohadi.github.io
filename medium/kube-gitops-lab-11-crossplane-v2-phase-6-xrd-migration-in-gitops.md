# Crossplane v2 Phase 6: Migrating LegacyCluster XRD to Namespaced XRD in GitOps

*Kube GitOps Lab — Part 11*

---

In Part 10, I closed the remaining gaps in the Crossplane v2 migration: `ClusterProviderConfig`, `ManagedResourceActivationPolicy`, and the first real `Cluster` claims. The closing promise was real cloud provisioning — actual credentials, actual infrastructure, actual nodes.

That promise is still coming. But before I could safely hand Crossplane the keys to a real cloud account, I hit one more structural problem: the XRD `scope` was wrong and it was immutable. This post is about fixing that — Phase 6 of the migration, using the **Orphan and Re-adopt** strategy to migrate from `LegacyCluster` scope (claim-based, v1 API) to `Namespaced` scope (direct XR, v2 API) without destroying existing resources.

You can follow along with the full source at [https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-3-crossplane-v2-migration-phase-6](https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-3-crossplane-v2-migration-phase-6).

---

### The Problem: XRD Scope Is Immutable

The Crossplane v2 path for composite resources is `Namespaced` XRs — you create an `XCluster` directly in the developer's namespace, no `Cluster` claim required. The claim-based flow uses `LegacyCluster` scope, which routes through a namespace-scoped Claim to a cluster-scoped XR.

When I set up the original lab in Part 5, I used `scope: LegacyCluster` in the XRD because it matched how Crossplane v1 worked. Now that the stack is on v2.2.0, the right scope is `Namespaced`. The problem: Kubernetes does not allow you to change `scope` on a live XRD. There is no patch path. You cannot `kubectl apply` a changed XRD and have it work.

So the options are:
1. Delete the XRD (and everything it owns) and recreate it — destructive if not careful
2. Orphan the managed resources first, then delete and recreate — safe

I went with option 2.

---

### Strategy: Orphan and Re-Adopt

The "Orphan and Re-adopt" pattern in Crossplane is a six-step sequence:

**1. Set management policies to omit `Delete`**

In Crossplane v2, the right way to orphan managed resources is `managementPolicies` — not `deletionPolicy: Orphan`. In v2, `deletionPolicy: Orphan` is silently ignored and your resources can be deleted when the XR goes away. Always use `managementPolicies` and omit `"Delete"`:

```yaml
spec:
  managementPolicies: ["Observe", "Create", "Update"]
```

Omitting `"Delete"` means Crossplane will not touch the external resource when the MR is deleted. It just walks away.

**2. Record the external names**

Crossplane identifies real-world resources via an annotation. Before deleting anything, capture every `crossplane.io/external-name` on your managed resources:

```bash
kubectl get managed -o custom-columns=\
KIND:.kind,\
NAME:.metadata.name,\
EXTERNAL:.metadata.annotations."crossplane\.io/external-name"
```

Save these. They are how the new XR will re-adopt the existing resources.

**3. Delete the Claims and the XRD**

Because of the orphan policy, deleting the Claims and then the XRD leaves the actual managed resources untouched. No cloud infrastructure is destroyed.

**4. Create the new v2 Namespaced XRD**

Apply the new XRD with `scope: Namespaced`, using `apiextensions.crossplane.io/v2`, and without a `claimNames` block.

**5. Re-adopt via a new Namespaced XR**

Create the new XR directly in the developer's namespace. Set `spec.parameters.externalName` to the value you recorded in step 2. The Composition patches this into `crossplane.io/external-name` on the managed resource, and the provider recognizes the existing resource rather than creating a new one.

**6. Verify**

The managed resources should transition to `SYNCED: True, READY: True` without any new external resources being created.

---

### Building the Safety Net: Legacy XRD and Composition

Before touching the main XRD, I needed to keep the existing claims alive during the transition. The current claims (`demo-cluster` and `demo-cluster-2`) were created against `infrastructure.lab.local/v1alpha1`. If I delete that XRD immediately, those claims become orphaned in Kubernetes — no schema, no reconciliation, potentially errors.

The solution: create a parallel `legacy.lab.local` XRD that mirrors the old structure exactly, then migrate the existing claims to point at it. This gives the claims a home while I rework the main XRD.

```yaml
# crossplane/compositions/cluster-xrd-legacy.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xclusters.legacy.lab.local
spec:
  scope: LegacyCluster
  group: legacy.lab.local
  names:
    kind: XCluster
    plural: xclusters
  claimNames:
    kind: Cluster
    plural: clusters
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    environment:
                      type: string
                      enum: [dev, staging, production]
                      default: dev
                    version:
                      type: string
                      default: "1.30"
                    nodeCount:
                      type: integer
                      default: 1
                      minimum: 1
                      maximum: 10
                  required:
                    - environment
```

This is identical to the old `cluster-xrd.yaml`, just under a new group. The legacy composition (`cluster-composition-legacy.yaml`) references this XRD:

```yaml
spec:
  compositeTypeRef:
    apiVersion: legacy.lab.local/v1alpha1
    kind: XCluster
  mode: Pipeline
  pipeline:
    - functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
          - base:
              apiVersion: kubernetes.m.crossplane.io/v1alpha1
              kind: Object
              spec:
                # managementPolicies: ["Observe", "Create", "Update"]
                forProvider:
                  manifest:
                    apiVersion: v1
                    kind: ConfigMap
                    metadata:
                      namespace: crossplane-system
                    data:
                      managed-by: crossplane
                      status: provisioned
                providerConfigRef:
                  name: default
                  kind: ClusterProviderConfig
            name: cluster-info
            patches:
              - fromFieldPath: metadata.name
                toFieldPath: spec.forProvider.manifest.metadata.name
                transforms:
                  - string:
                      fmt: cluster-info-%s
                      type: Format
                    type: string
                type: FromCompositeFieldPath
              - fromFieldPath: spec.parameters.environment
                toFieldPath: spec.forProvider.manifest.data.environment
                type: FromCompositeFieldPath
              - fromFieldPath: spec.parameters.version
                toFieldPath: spec.forProvider.manifest.data.kubernetes-version
                type: FromCompositeFieldPath
              - fromFieldPath: spec.claimRef.namespace
                toFieldPath: metadata.namespace
                type: FromCompositeFieldPath
          step: patch-and-transform
```

The legacy composition keeps the `spec.claimRef.namespace` patch because claims are still namespace-scoped under `LegacyCluster`. This is the old wiring, preserved intentionally for the claims that are temporarily parked here.

---

### Migrating Claims to Legacy

With the legacy XRD and composition in place, I updated both existing claims to use the legacy API group:

```yaml
# crossplane/claims/demo-cluster.yaml
apiVersion: legacy.lab.local/v1alpha1  # was: infrastructure.lab.local/v1alpha1
kind: Cluster
metadata:
  name: demo-cluster
  namespace: demo
spec:
  parameters:
    environment: dev
    version: "1.31"
    nodeCount: 1
```

```yaml
# crossplane/claims/demo-cluster-2.yaml
apiVersion: legacy.lab.local/v1alpha1  # was: infrastructure.lab.local/v1alpha1
kind: Cluster
metadata:
  name: demo-cluster-2
  namespace: demo
spec:
  parameters:
    environment: staging
    version: "1.31"
    nodeCount: 2
```

Crossplane re-binds the claims to the new XRD. The managed resources are unaffected — same ConfigMaps, same external names, same state.

---

### Upgrading the Main XRD to v2 Namespaced

With the claims safely parked under `legacy.lab.local`, I updated `cluster-xrd.yaml`:

```yaml
apiVersion: apiextensions.crossplane.io/v2    # was: v1
kind: CompositeResourceDefinition
metadata:
  name: xclusters.infrastructure.lab.local
spec:
  scope: Namespaced                            # was: LegacyCluster
  group: infrastructure.lab.local
  names:
    kind: XCluster
    plural: xclusters
  # claimNames removed — no claims in v2 Namespaced XRs
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    environment:
                      type: string
                      enum: [dev, staging, production]
                      default: dev
                    version:
                      type: string
                      default: "1.30"
                    nodeCount:
                      type: integer
                      default: 1
                      minimum: 1
                      maximum: 10
                    externalName:             # new field for adoption
                      type: string
                      description: "If set, adopts existing resource. If empty, creates new."
                  required:
                    - environment
```

Three changes from the previous version:

- **`apiVersion: apiextensions.crossplane.io/v2`** — required for v2 XRD features and scoping
- **`scope: Namespaced`** — XRs now live directly in developer namespaces, no claim intermediary
- **`claimNames` removed** — in Namespaced scope there are no claims, so the block is gone
- **`externalName` field added** — this is the adoption hook; if a value is provided, the Composition patches it onto the managed resource annotation

The `externalName` field is optional in the schema (`required` only lists `environment`), so existing XRs that don't need adoption simply omit it.

---

### Wiring the Composition for Adoption

The main `cluster-composition.yaml` needed one change: replace the namespace patch with an `externalName` adoption patch.

The old patch read the claim's namespace and applied it to the composed Object's namespace — that was claim-scoped wiring. In a Namespaced XR there is no `spec.claimRef`, so that patch fails. The new patch reads the optional `externalName` field and pushes it into the `crossplane.io/external-name` annotation:

```yaml
# Patch the Real Name for Adoption
- fromFieldPath: spec.parameters.externalName
  toFieldPath: metadata.annotations[crossplane.io/external-name]
  policy:
    fromFieldPath: Required  # Only runs if user provides the name
  type: FromCompositeFieldPath
```

The `fromFieldPath: Required` policy is the key detail here. If `spec.parameters.externalName` is empty or missing, this patch is skipped entirely — Crossplane creates a new resource with a generated name. If the field has a value, the patch runs and the provider looks for an existing resource matching that external name instead of creating a new one.

This is the adoption mechanism. The provider sees the annotation, checks for an existing resource, and if it finds one it begins managing it rather than creating a duplicate.

---

### Creating Composite Resources Directly

With the new Namespaced XRD in place, I created the replacement XRs without going through claims:

```yaml
# crossplane/composite-resources/demo-cluster.yaml
apiVersion: infrastructure.lab.local/v1alpha1
kind: XCluster
metadata:
  name: demo-cluster-84lfb
  namespace: demo
spec:
  parameters:
    environment: dev
    version: "1.31"
    nodeCount: 1
    # This triggers the adoption
    externalName: demo-cluster-84lfb-c3606e9493a5
```

```yaml
# crossplane/composite-resources/demo-cluster-2.yaml
apiVersion: infrastructure.lab.local/v1alpha1
kind: XCluster
metadata:
  name: demo-cluster-2
  namespace: demo
spec:
  parameters:
    environment: staging
    version: "1.31"
    nodeCount: 2
    externalName: demo-cluster-2-l7wh7
```

A few things worth noting:

- **`namespace: demo`** — the XR lives directly in the `demo` namespace, the same namespace where the developer works. No claim object sits between the user and the XR.
- **`externalName`** — these values are the names of the existing ConfigMaps that the old claims created. I captured them via `kubectl get managed` before the migration. The Composition patches each value into `crossplane.io/external-name` on the composed Object, so the kubernetes provider recognizes the existing ConfigMap and adopts it rather than creating a second one.
- **`version: "1.31"`** — bumped from 1.30. The XR schema update is GitOps-tracked just like everything else.

---

### ArgoCD Branch Wiring

Following the same pattern from Parts 7, 8, 9, and 10, I bumped `targetRevision` in three ArgoCD Application manifests:

- `apps/08-crossplane-providers.yaml`
- `apps/09-crossplane-providerconfigs.yaml`
- `apps/10-crossplane-compositions.yaml`

All three moved from `feat/phase-3-crossplane-v2-migration-phase-5` to `feat/phase-3-crossplane-v2-migration-phase-6`. ArgoCD detects the change, syncs in wave order, and the new XRD, legacy XRD, and updated Composition deploy automatically.

The composite resources under `crossplane/composite-resources/` are applied directly — same as the claims, they represent workload intent and live outside the ArgoCD app boundary.

---

### What Changed

- **`cluster-xrd-legacy.yaml`** — new XRD mirroring the old `LegacyCluster` structure under `legacy.lab.local`; keeps existing claims alive during migration
- **`cluster-composition-legacy.yaml`** — new Composition for the legacy claim-based flow; preserves namespace patching from claim
- **Claims updated** — `demo-cluster` and `demo-cluster-2` moved to `legacy.lab.local/v1alpha1` as a temporary safe harbor
- **`cluster-xrd.yaml`** — upgraded to `apiextensions.crossplane.io/v2`, `scope: Namespaced`, `claimNames` removed, `externalName` field added
- **`cluster-composition.yaml`** — namespace patch replaced with `externalName` adoption patch using `policy.fromFieldPath: Required`
- **`composite-resources/demo-cluster.yaml`** and **`composite-resources/demo-cluster-2.yaml`** — new direct XRs in the `demo` namespace, with `externalName` values for re-adoption
- **ArgoCD apps** — branch bumped to phase-6 across `apps/08`, `apps/09`, `apps/10`

The key takeaway from this phase: the `externalName` annotation is how Crossplane providers locate and re-adopt existing resources. Whether the external resource is a ConfigMap in a local cluster or an S3 bucket in AWS, the mechanism is the same — set the right value on the annotation and the provider takes over management rather than creating a duplicate. Note that not every provider respects this equally; always verify against the provider docs and test in a staging environment before applying this pattern to production.

---

### What's Next

With `scope: Namespaced` in place and XRs reconciling correctly under the v2 API, the Crossplane migration is structurally complete. The lab setup now matches how Crossplane v2 is designed to work.

The original goal from Part 5 — real multi-cluster provisioning with a real cloud provider — is finally within reach. Next up: wiring Crossplane to an actual cloud account, adding provider credentials that matter, and watching `demo-cluster` provision something you can `kubectl` into.

Stay tuned for next stories with The Hands-On Engineer!

---

*Tags: Kubernetes, GitOps, Crossplane, ArgoCD, DevOps, Platform Engineering, Cloud Native*
