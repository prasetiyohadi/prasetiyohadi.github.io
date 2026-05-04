# Kube GitOps Lab 05: Adding Multi-Cluster Provisioning with Crossplane in a GitOps Lab

In the [previous post](https://medium.com/), I completed the security stack on my single-node k3s management cluster: Trivy Operator for image vulnerability scanning, and Kubescape for compliance posture management. Both run continuously and feed results back into the cluster via CRDs, no dashboards required.

That gave me a fully observable, policy-enforced cluster. But everything I built so far manages workloads on a single cluster. Phase 3 is about going further: using the management cluster to provision and manage other clusters. That's the promise of Crossplane.

In this post, I walk through how I integrated Crossplane into the GitOps setup using ArgoCD sync waves, why the wave ordering matters more than you'd think, and what the XRD + Composition abstraction looks like even when you're running locally without a cloud provider.

The source code is on GitLab: [pras-labs/kube-gitops-demo](https://gitlab.com/pras-labs/kube-gitops-demo)

---

### What Crossplane Does (and Why I Want It Here)

Crossplane is a Kubernetes add-on that lets you manage infrastructure using standard Kubernetes APIs. Instead of running Terraform or clicking through a cloud console, you define cloud resources as YAML manifests and apply them to your cluster. Crossplane's controllers reconcile those manifests against the real world.

The core building blocks:

- **Providers** — controllers that know how to talk to a specific system (AWS, GCP, Helm, Kubernetes, etc.)
- **ProviderConfigs** — credentials and connection settings for a Provider
- **CompositeResourceDefinitions (XRDs)** — custom API types you define (e.g., `XCluster`)
- **Compositions** — the implementation behind an XRD; defines what real resources get created when someone claims an `XCluster`

The management cluster pattern I'm building toward: ArgoCD deploys everything via GitOps, and Crossplane provisions new clusters on demand. New cluster request → git commit → ArgoCD syncs → Crossplane reconciles → real cluster appears. No manual cloud console interaction.

For now, I'm running a local k3s lab with no cloud provider. So this post focuses on wiring Crossplane into the GitOps structure correctly, and demonstrating the abstraction pattern with a stand-in for the real thing.

---

### The CRD Race Condition Problem

Before I get into the manifests, here's the problem that forced a specific deployment order.

Crossplane's providers each install their own CRDs. For example, the Kubernetes provider installs the `ProviderConfig` CRD in the `kubernetes.crossplane.io` API group. If ArgoCD tries to sync a `kubernetes.crossplane.io/v1alpha1 ProviderConfig` manifest before that CRD exists, the sync fails.

The naive approach is one ArgoCD app pointing at the whole `crossplane/` directory. But that creates a race: ArgoCD applies everything at once, and the ProviderConfig lands before its CRD does.

The solution I used: split the deployment into four separate ArgoCD apps across four sync waves.

- **Wave 7**: Crossplane core (Helm chart); installs the control plane and base CRDs
- **Wave 8**: Provider manifests; installs the Helm and Kubernetes providers, which register their own CRDs
- **Wave 9**: ProviderConfigs; now the CRDs from Wave 8 exist, so these can apply cleanly
- **Wave 10**: XRDs and Compositions; our custom abstractions, applied last

ArgoCD's sync wave mechanism means each wave waits for the previous wave's resources to reach a healthy state before proceeding. That's what makes this work without any manual intervention.

---

### Wave 7: Installing Crossplane Core

```yaml
# apps/07-crossplane.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "7"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://charts.crossplane.io/stable
    chart: crossplane
    targetRevision: 1.19.0

    helm:
      values: |
        replicas: 1

        resourcesCrossplane:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi

        resourcesRBACManager:
          requests:
            cpu: 50m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi

        metrics:
          enabled: false

  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 30s
        factor: 2
        maxDuration: 5m
```

Single-node k3s adjustments I made vs. defaults:

- **`replicas: 1`**; default Crossplane deployment wants 1 replica anyway, but being explicit prevents any future template changes from adding replicas unexpectedly
- **`metrics.enabled: false`**; I don't have Prometheus yet; no point scraping nothing
- **Resource limits** — scaled down for a lab node; Crossplane's default requests are generous

This app installs into `crossplane-system` (created automatically via `CreateNamespace=true`) and brings up the Crossplane controller and RBAC manager.

---

### Wave 8: Installing the Providers

With Crossplane core healthy, Wave 8 installs the Provider CRs. I'm using the Helm and Kubernetes providers from `crossplane-contrib`, these let Crossplane manage Helm releases and raw Kubernetes objects as managed resources.

```yaml
# crossplane/providers/manifests/helm-provider.yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: upbound-provider-helm
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-helm:v0.19.0
  runtimeConfigRef:
    name: default
```

```yaml
# crossplane/providers/manifests/kubernetes-provider.yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: upbound-provider-kubernetes
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.15.0
  runtimeConfigRef:
    name: default
```

The ArgoCD app that syncs these is Wave 8, pointing at the `crossplane/providers/manifests/` path:

```yaml
# apps/08-crossplane-providers.yaml (excerpt)
annotations:
  argocd.argoproj.io/sync-wave: "8"
source:
  repoURL: https://gitlab.com/pras-labs/kube-gitops-demo.git
  targetRevision: feat/phase-3-initial-crossplane-multi-cluster-provisioning
  path: crossplane/providers/manifests/
```

When Crossplane sees a `Provider` CR, it pulls the package image and runs an OCI-packaged controller. Those controllers register their CRDs into the cluster. That's what Wave 9 depends on.

---

### Wave 9: ProviderConfigs

ProviderConfigs tell each provider how to authenticate. In a real setup, you'd reference a Secret containing cloud credentials or a kubeconfig for a remote cluster. In my local k3s lab, `InjectedIdentity` is the right choice; it tells the provider to use the management cluster's own service account token.

```yaml
# crossplane/providers/configs/kubernetes-providerconfig.yaml
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: InjectedIdentity
```

```yaml
# crossplane/providers/configs/helm-providerconfig.yaml
apiVersion: helm.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: InjectedIdentity
```

Both use `InjectedIdentity` and name themselves `default`, which is what Compositions will reference unless you specify otherwise.

The Wave 9 ArgoCD app has one key difference from Wave 8, it does **not** use `CreateNamespace=true`. These resources go into `crossplane-system`, which already exists from Wave 7.

```yaml
# apps/09-crossplane-providerconfigs.yaml (excerpt)
annotations:
  argocd.argoproj.io/sync-wave: "9"
syncPolicy:
  syncOptions:
    - ServerSideApply=true
```

`ServerSideApply=true` is set in both Wave 8 and Wave 9. Crossplane Provider and ProviderConfig resources update frequently as the controllers reconcile state, and server-side apply handles the merge correctly without ownership conflicts.

---

### Wave 10: XRDs and Compositions

This is where Crossplane's abstraction power shows up. An XRD (`CompositeResourceDefinition`) lets you define your own Kubernetes resource type. A Composition implements it, mapping your custom type to real managed resources.

```yaml
# crossplane/compositions/cluster-xrd.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xclusters.infrastructure.lab.local
spec:
  group: infrastructure.lab.local
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

This defines `XCluster` as a new API type with three parameters: `environment`, `version`, and `nodeCount`. Anyone on the platform can create a `Cluster` claim (the namespaced variant) without knowing anything about the underlying infrastructure implementation.

The Composition that backs it:

```yaml
# crossplane/compositions/cluster-composition.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: cluster-local-lab
  labels:
    provider: local
    environment: lab
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

In a real cloud setup, the `resources` array would contain VPC definitions, managed cluster resources (EKS, GKE, AKS), node group configurations, and then Helm releases to install Cilium and ArgoCD on the new cluster; all wired together with patches that flow the `XCluster` parameters down to each resource.

Here, the Composition creates a ConfigMap as a proof-of-concept. It demonstrates the patch mechanism: `metadata.name` flows into the ConfigMap's name, `spec.parameters.environment` flows into a ConfigMap data field. The structure is identical to what a real Composition would use — only the managed resource type changes.

That's the honest trade-off of running this in a local k3s lab. The wiring is real; the cloud resources aren't.

---

### Conclusion

Phase 3 adds Crossplane as the multi-cluster provisioning layer on top of the GitOps foundation I've been building. Here's what went in:

- **Crossplane Core** installed via Helm in its own ArgoCD Application (Wave 7), tuned for single-node k3s
- **Helm and Kubernetes Providers** deployed as separate ArgoCD Applications (Wave 8) to let them install their CRDs before ProviderConfigs land
- **ProviderConfigs** in Wave 9 using `InjectedIdentity`, no secrets needed for local cluster management
- **XRD and Composition** in Wave 10 defining the `XCluster` abstraction with a ConfigMap stand-in for cloud resources

The most important takeaway: the wave ordering isn't arbitrary. It directly solves the CRD race condition that makes "just apply everything at once" fail for Crossplane. Splitting providers and providerconfigs into separate apps with explicit wave annotations is the pattern that makes GitOps + Crossplane reliable.

The placeholder Composition is also worth noting. It's not a hack — it's a complete demonstration of the patch mechanism that a real cloud Composition uses. When I add real cloud provider credentials, the Composition implementation changes but the API (`XCluster`) stays the same. That's the abstraction working as intended.

---

### What's Next

Phase 3 establishes the structure. The next logical step is replacing the ConfigMap placeholder with actual cloud provider resources; probably an AWS provider with EKS cluster provisioning, showing how the `XCluster` XRD maps to real VPC and managed cluster resources.

Stay tuned for next stories with Pras' Labs: The Hands-On Engineer!

---

_Tags: Kubernetes, GitOps, Crossplane, ArgoCD, DevOps, Platform Engineering, Multi-Cluster, Infrastructure as Code_
