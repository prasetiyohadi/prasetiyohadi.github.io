# Kube GitOps Lab 10: Crossplane v2 Phase 5: Post-Migration Hardening

---

In the [previous post](https://medium.com/@prasetiyohadi/crossplane-v2-phase-4-upgrading-providers-and-adopting-namespace-scoped-objects-kubernetes-gitops), I upgraded `provider-helm` and `provider-kubernetes` to their official v2-era releases, migrated the Composition from the legacy `kubernetes.crossplane.io` API to the modern `kubernetes.m.crossplane.io` API, and hit the `spec.scope` immutability wall head-on. By the end of that post, the migration was architecturally complete. I wrapped it up by promising the next logical step: real cloud credentials, real XCluster claims, something you can actually SSH into.

Then I went to test it.

There were three gaps I needed to close before that promise made sense: the v2 providers needed a `ClusterProviderConfig` (not just a `ProviderConfig`), the Composition needed to be told to use it, and the composed Objects needed a namespace patch so they'd land in the right place. I also added a `ManagedResourceActivationPolicy` to stop Crossplane from watching every CRD in the cluster. This post covers all of that: Phase 5, post-migration hardening.

Repository: [gitlab.com/pras-labs/kube-gitops-demo](https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-3-crossplane-v2-migration-phase-5)

---

### ClusterProviderConfig: The v2 Scoping Fix

When Crossplane v2 introduced the `.m.crossplane.io` provider APIs (`kubernetes.m.crossplane.io`, `helm.m.crossplane.io`), it also changed how provider configs are scoped. The old `ProviderConfig` kind is namespace-scoped (legacy, "LegacyCluster" in v2 terminology). The new kind is `ClusterProviderConfig`, genuinely cluster-scoped in the v2 sense.

If your Composition still references a `ProviderConfig` but your composed resources use `.m.crossplane.io` types, Crossplane will look for the wrong kind and fail silently or produce confusing errors. I needed to create `ClusterProviderConfig` resources for both providers.

**`crossplane/providers/configs/kubernetes-clusterproviderconfig.yaml`:**

```yaml
# ClusterProviderConfig: use in-cluster credentials
# V2 cluster scoped
apiVersion: kubernetes.m.crossplane.io/v1alpha1
kind: ClusterProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: InjectedIdentity
```

**`crossplane/providers/configs/helm-clusterproviderconfig.yaml`:**

```yaml
# ClusterProviderConfig: use in-cluster credentials (management cluster's service account)
# V2 cluster scoped
apiVersion: helm.m.crossplane.io/v1beta1
kind: ClusterProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: InjectedIdentity
```

Both use `InjectedIdentity`, the in-cluster service account that Crossplane already has. No new secrets, no credentials to manage. The only change from the existing `ProviderConfig` resources is the `apiVersion` (`.m.crossplane.io`) and the `kind` (`ClusterProviderConfig`).

I updated the comments on the existing `ProviderConfig` resources to make the distinction explicit:

```yaml
# ProviderConfig: use in-cluster credentials
# V1 LegacyCluster scoped   <- was just "Cluster scoped"
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
```

The old `ProviderConfig` resources stay. They're still referenced by any legacy resources in the cluster. The new `ClusterProviderConfig` resources are what the updated Composition will use.

---

### Updating the Composition

With the `ClusterProviderConfig` resources in place, I needed to update the Composition to actually reference them. Crossplane defaults to looking up `ProviderConfig` when no `kind` is specified in `providerConfigRef`. Two changes were needed.

**First: explicit `kind` in `providerConfigRef`**

```yaml
providerConfigRef:
  name: default
  kind: ClusterProviderConfig # <- added
```

Without this, Crossplane resolves `providerConfigRef.name: default` against the `ProviderConfig` kind: the v1 legacy resource. Adding `kind: ClusterProviderConfig` tells it to resolve against the right type.

**Second: namespace patch for composed Objects**

```yaml
patches:
  - fromFieldPath: spec.claimRef.namespace
    toFieldPath: metadata.namespace
    type: FromCompositeFieldPath
```

This one bit me in testing. When a Claim is created in namespace `demo`, the XCluster composite is created at the cluster level, and the Composition renders the Object. Without a namespace patch, the Object ends up in `crossplane-system` by default. With this patch, the Object inherits the namespace from the Claim: `demo` in my case. That matters once real managed resources are involved, because RBAC and resource isolation are namespace-aware.

---

### MRAP: Targeted Resource Activation

Crossplane v2 ships with a concept called `ManagedResourceActivationPolicy` (MRAP). By default, v2 auto-creates a wildcard MRAP that activates all Managed Resource Definitions in the cluster. Crossplane then watches all of those types for changes. In a lab with a handful of providers, that's a few extra watches. In a cluster with many providers, it's wasted memory and CPU.

I added a targeted MRAP to `crossplane/providers/configs/mrap.yaml`:

```yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: ManagedResourceActivationPolicy
metadata:
  name: crossplane-resources
  namespace: crossplane-system
spec:
  activate:
    # Legacy cluster-scoped (v1 style)
    - objects.kubernetes.crossplane.io
    - releases.helm.crossplane.io
    # V2 namespaced - we only use these:
    - objects.kubernetes.m.crossplane.io
    - releases.helm.m.crossplane.io
```

Four resource types, two legacy and two v2. That's all this lab uses. Crossplane watches only those types and ignores everything else.

I placed this file in `crossplane/providers/configs/` so the existing ArgoCD app `09-crossplane-providerconfigs` picks it up automatically. No new ArgoCD Application, no wave changes; it's a config-layer concern and it belongs with the other provider configs.

---

### Verifying with Cluster Claims

At this point I had all the v2 wiring in place. The natural next test was to create actual `Cluster` claims and see whether the path: Claim -> XCluster -> Composition -> ClusterProviderConfig -> Object in claim namespace, worked end-to-end.

**`crossplane/claims/demo-cluster.yaml`:**

```yaml
apiVersion: infrastructure.lab.local/v1alpha1
kind: Cluster
metadata:
  name: demo-cluster
  namespace: demo
spec:
  parameters:
    environment: dev
    version: "1.30"
    nodeCount: 1
```

**`crossplane/claims/demo-cluster-2.yaml`:**

```yaml
apiVersion: infrastructure.lab.local/v1alpha1
kind: Cluster
metadata:
  name: demo-cluster-2
  namespace: demo
spec:
  parameters:
    environment: staging
    version: "1.30"
    nodeCount: 1
```

Two claims: `demo-cluster` for dev, `demo-cluster-2` for staging. Both in the `demo` namespace, both targeting Kubernetes v1.30 with a single node.

In this lab setup, the Composition doesn't actually provision real cloud infrastructure; it creates a Kubernetes `Object` (a ConfigMap representing cluster-info) on the local cluster. That ConfigMap lands in the `demo` namespace now (not `crossplane-system`) because of the namespace patch from the previous section. The real cloud provisioning: actual credentials; actual cloud provider; actual nodes, comes in the next post. These claims are the smoke test: if they reconcile and the ConfigMap appears in the right place, the v2 wiring is correct.

---

### ArgoCD Branch Wiring

Following the same branch-per-phase pattern established in Parts 7 and 8, I bumped the `targetRevision` in three ArgoCD Application manifests from `feat/phase-3-crossplane-v2-migration-phase-4` to `feat/phase-3-crossplane-v2-migration-phase-5`:

- `apps/08-crossplane-providers.yaml`
- `apps/09-crossplane-providerconfigs.yaml` (picks up MRAP + new ClusterProviderConfigs)
- `apps/10-crossplane-compositions.yaml` (picks up updated Composition)

ArgoCD detects the branch change, syncs each app in wave order, and the new resources deploy without manual intervention. The claims don't live in an ArgoCD app; they're committed to the repo and applied directly, which is intentional: claims represent workload intent, not platform config.

---

### What Changed

- **ClusterProviderConfig (kubernetes + helm)**: v2 cluster-scoped provider config resources using in-cluster credentials
- **Composition update**: explicit `kind: ClusterProviderConfig` in `providerConfigRef`, plus namespace patch to place composed Objects in the claim's namespace
- **MRAP**: targeted `ManagedResourceActivationPolicy` activating only the four resource types the lab actually uses
- **Cluster claims**: first real `Cluster` claims (`demo-cluster` and `demo-cluster-2`) as an end-to-end smoke test

The practical takeaway from this whole migration series is that v2 isn't a single upgrade. It's a sequence of gaps to close: version hops in Parts 6-8, provider APIs in Part 9, and now scoping and activation policies in Part 10. Each phase is small. Each one builds on the last.

---

### What's Next

The foundation is now solid. Crossplane is on v2.2.0, providers are on their latest releases, the Composition uses v2 APIs with correct scoping, and claims reconcile correctly. The original goal from Part 5 was always real multi-cluster provisioning; an XCluster claim that actually provisions infrastructure somewhere you can connect to.

That's what comes next: wiring up a real cloud provider, giving Crossplane credentials that matter, and watching `demo-cluster` become something more than a ConfigMap.

Stay tuned for next stories with Pras' Labs: The Hands-On Engineer!

---

_Tags: Kubernetes, GitOps, Crossplane, ArgoCD, DevOps, Platform Engineering, Infrastructure as Code_
