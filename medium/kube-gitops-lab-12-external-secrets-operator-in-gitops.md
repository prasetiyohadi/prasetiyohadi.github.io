# External Secrets Operator: The Credential Bridge for Crossplane — Kubernetes GitOps Lab, Part 12

*Kube GitOps Lab — Part 12*

---

In Part 11, I completed the Crossplane v2 XRD migration — `scope: Namespaced`, `externalName` adoption for existing composite resources, and the full pipeline-mode Composition rewrite. The migration branch, which had been accumulating phases since Part 6, is now merged to `main`. The Crossplane v2 foundation is stable.

The original goal from Part 5 was real multi-cluster provisioning with a real cloud provider. I've been working toward that ever since, and now that the migration is done, the next blocker is credentials. Crossplane ProviderConfigs need a Kubernetes Secret with real cloud credentials, and I'm not putting those in git. That's where External Secrets Operator comes in. This post covers installing ESO as the credential management layer — the piece that sits between a real secrets store and the K8s Secrets that Crossplane will consume.

The full source is on GitLab: [kube-gitops-demo @ feat/phase-4-setup-external-secrets-operator](https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-4-setup-external-secrets-operator)

---

### Closing the Migration Branch

The first change in this diff is the quietest but most meaningful. Three ArgoCD Application manifests — `apps/08-crossplane-providers.yaml`, `apps/09-crossplane-providerconfigs.yaml`, and `apps/10-crossplane-compositions.yaml` — had been tracking a feature branch throughout the migration:

```yaml
# Before
targetRevision: feat/phase-3-crossplane-v2-migration-phase-6

# After
targetRevision: main
```

That feature branch was the workspace for the entire Crossplane v2 migration — six phases spanning Parts 6 through 11. Moving `targetRevision` back to `main` means the migration is merged and the GitOps state is now canonical. ArgoCD will no longer diverge from the main branch if someone rebases or cleans up the feature branch.

It's a one-line change per file, but it closes a chapter. From here, the Crossplane apps track `main` like everything else in the cluster.

---

### Why ESO Before Crossplane Credentials

Crossplane ProviderConfigs reference a Kubernetes Secret that holds cloud credentials. For AWS, that's an access key and secret. For GCP, it's a service account JSON. The Secret has to exist in the cluster before the ProviderConfig syncs, or the provider goes into an error state.

The naive approach is to create that Secret manually with `kubectl create secret`. That works, but it breaks the GitOps model: the Secret isn't tracked in the repository, it won't survive a cluster rebuild, and whoever ran the command holds the credentials in their shell history.

External Secrets Operator solves this by acting as a bridge. Instead of storing credentials in git, I store a reference — which secret store to look in, which path to read. ESO holds the actual credentials in AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager, or whichever backend I choose. At sync time, ESO reads from the external store and writes a native Kubernetes Secret into the cluster. The credential never touches git; the git manifest only describes where to find it.

Once ESO is running, I can create an `ExternalSecret` resource that points at, say, an AWS Secrets Manager path like `prod/crossplane/aws-credentials`. ESO fetches it and materializes a K8s Secret with the right keys. The Crossplane ProviderConfig then references that Secret — and Crossplane can finally talk to a real cloud account.

That's the setup order: ESO first, then SecretStore + ExternalSecret, then ProviderConfig. This post covers the first step.

---

### The ArgoCD Application — Wave 11

ESO lives in its own ArgoCD Application at `apps/11-external-secrets-operator.yaml`, synced at wave 11 — after the Crossplane operators and providers are up:

```yaml
# External Secrets Operator: Wave 11 (Secret Sync)
#
# Syncs secrets from external stores (AWS SM, Vault, GCP SM, etc.) into
# native Kubernetes Secret objects. Prerequisite for Crossplane providers
# and workloads that need real credentials instead of InjectedIdentity.
#
# Single-node lab adjustments:
#   - replicaCount: 1 (controller, webhook, cert-controller)
#   - serviceMonitor.enabled: false
#
# Chart: https://charts.external-secrets.io
# Version: 2.3.0

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets-operator
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "11"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://charts.external-secrets.io
    chart: external-secrets
    targetRevision: 2.3.0

    helm:
      values: |
        installCRDs: true

        replicaCount: 1

        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 256Mi

        serviceMonitor:
          enabled: false

        webhook:
          replicaCount: 1
          serviceMonitor:
            enabled: false
          resources:
            requests:
              cpu: 25m
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 128Mi

        certController:
          replicaCount: 1
          serviceMonitor:
            enabled: false
          resources:
            requests:
              cpu: 25m
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 128Mi

  destination:
    server: https://kubernetes.default.svc
    namespace: external-secrets

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
    retry:
      limit: 5
      backoff:
        duration: 30s
        factor: 2
        maxDuration: 5m
```

A few things worth calling out:

- **`installCRDs: true`** — ESO ships its own CRDs: `SecretStore`, `ClusterSecretStore`, `ExternalSecret`, `ClusterExternalSecret`, and others. Setting this in Helm values is the simplest way to get them installed alongside the operator without a separate ArgoCD app for CRD management.

- **Three components, all at replicaCount: 1** — ESO deploys a controller, a webhook, and a cert-controller. In a production HA setup you'd run multiples of each. In a single-node lab, 1 of each is fine and saves around 200m CPU and 400Mi memory in idle.

- **`serviceMonitor: false`** — I'm not running Prometheus in this lab setup, so the ServiceMonitor objects would just be orphaned resources. Disabled across all three components.

- **`ServerSideApply=true`** — ESO's CRDs have complex validation schemas. Server-side apply handles the field ownership tracking that client-side apply struggles with on large CRD objects. This avoids annotation bloat and the `metadata.annotations too long` errors I've hit with other operators.

- **Retry with exponential backoff** — ESO's webhook needs to be healthy before other resources can use ESO CRDs. The retry (limit 5, 30s base, factor 2, max 5 minutes) gives cert-controller time to issue the webhook's TLS certificate before ArgoCD gives up.

---

### The ignoreDifferences Trick for cert-controller

ESO uses its own internal `cert-controller` to manage TLS for the admission webhook. After every reconcile cycle, cert-controller rewrites the `caBundle` field on the CRD conversion webhook configuration and the admission webhook configurations. This is by design — it's how cert-controller keeps the CA bundle fresh.

The problem: ArgoCD reads those fields, sees they differ from what's in git (because there's nothing in git for a dynamically-generated CA bundle), and flags the application as out-of-sync. Without intervention, ArgoCD and cert-controller fight each other in a loop — ArgoCD reverts, cert-controller rewrites, repeat.

The fix is `ignoreDifferences`:

```yaml
  ignoreDifferences:
    - group: apiextensions.k8s.io
      kind: CustomResourceDefinition
      jsonPointers:
        - /spec/conversion/webhook/clientConfig/caBundle
    - group: admissionregistration.k8s.io
      kind: ValidatingWebhookConfiguration
      jqPathExpressions:
        - '.webhooks[].clientConfig.caBundle'
    - group: admissionregistration.k8s.io
      kind: MutatingWebhookConfiguration
      jqPathExpressions:
        - '.webhooks[].clientConfig.caBundle'
```

Three entries, two different path syntaxes:

- **`jsonPointers`** — used for the CRD, where the path is a single deterministic field at a known location. JSON Pointer syntax (`/spec/conversion/...`) is exact and unambiguous.
- **`jqPathExpressions`** — used for the webhook configurations, where `caBundle` appears inside an array of webhook entries. `jqPathExpressions` supports array traversal; `jsonPointers` does not.

This pattern shows up whenever a controller manages its own TLS — cert-manager, cert-controller, or any operator with a self-signed admission webhook. Once I understood that the loop is caused by dynamic CA injection, the fix is always the same: tell ArgoCD to ignore those specific fields. I'll reuse this in future posts.

---

### What Changed

- **`apps/08-crossplane-providers.yaml`**, **`apps/09-crossplane-providerconfigs.yaml`**, **`apps/10-crossplane-compositions.yaml`** — `targetRevision` moved from migration feature branch to `main`; GitOps state is now canonical
- **`apps/11-external-secrets-operator.yaml`** — new ArgoCD Application installing External Secrets Operator via Helm chart v2.3.0 at wave 11
- **ESO components**: controller, webhook, cert-controller — all at replicaCount: 1, serviceMonitor disabled, resource-budgeted for single-node
- **`ignoreDifferences`** on three resource types to suppress cert-controller CA bundle drift

Tools in this post:
- **ArgoCD** — GitOps sync engine, wave-ordered installation
- **External Secrets Operator** — secret sync bridge between external stores and K8s Secrets
- **Helm** — chart delivery for ESO (v2.3.0 from `charts.external-secrets.io`)
- **cert-controller** — ESO's internal TLS manager for the admission webhook

The `ignoreDifferences` pattern for cert-controller-managed webhooks is reusable across any operator that injects its own CA bundle. The three-entry structure (CRD + ValidatingWebhookConfiguration + MutatingWebhookConfiguration) covers the full surface area.

---

### What's Next

ESO is deployed and running. The next step is to give it somewhere to read from. I'll configure a `SecretStore` (or `ClusterSecretStore`) pointing at a real secrets backend — most likely AWS Secrets Manager to pair with the AWS provider I'll be wiring into Crossplane. Then I'll create an `ExternalSecret` that maps the AWS SM path to a K8s Secret in the `crossplane-system` namespace. Once that Secret exists, the Crossplane ProviderConfig can reference it — and an `XCluster` claim can finally provision something real.

Stay tuned for next stories with The Hands-On Engineer!

---

*Tags: Kubernetes, GitOps, External Secrets, Crossplane, ArgoCD, DevOps, Platform Engineering, Cloud Native*
