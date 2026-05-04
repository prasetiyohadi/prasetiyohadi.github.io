# Kube GitOps Lab 02: Adding Kyverno Admission Control and Policies via ArgoCD

In the [previous post](https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-1-bootstrap-k8s-cilium-argocd-apps) I set up a single-node `k3s` cluster, installed Cilium manually to break the CNI bootstrap deadlock, and then let ArgoCD adopt Cilium and manage it from that point forward. That "adopt-then-manage" pattern is the foundation. From here, everything new gets deployed cleanly through ArgoCD with no manual `kubectl apply` on the live cluster.

This post covers Phase 2b: Kyverno as an admission controller and four baseline ClusterPolicies deployed via ArgoCD. No bootstrap tricks this time, Kyverno goes in entirely through GitOps because Cilium is already running and pod networking works.

The code for this phase is here: [https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-2b-initial-kyverno-policies](https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-2b-initial-kyverno-policies)

---

### Why Admission Control Before Runtime Security

Admission control is the last gate before a pod hits the scheduler. If a workload violates a policy: wrong image tag; missing resource limits; privileged container, the admission controller rejects it at creation time. Runtime security tools like Falco catch bad behavior after the pod is already running. Both layers matter, but admission control is the right starting point: it is cheap to enforce and stops obvious misconfigurations before they land.

I picked Kyverno over the alternatives for one reason: policies are plain Kubernetes YAML. No Rego, no custom DSL, no separate policy engine to learn. You write a `ClusterPolicy` the same way you write any other Kubernetes resource, and ArgoCD manages it the same way it manages everything else. The entire policy lifecycle: write; review; apply; audit; enforce, runs through Git.

---

### The Repository Structure After Phase 2b

```bash
kube-gitops-demo
├── apps
│   ├── 00-cilium.yaml
│   ├── 01-kyverno.yaml
│   └── 02-kyverno-policies.yaml
├── bootstrap
│   ├── argocd-apps.yaml
│   └── argocd-install.yaml
├── policies
│   └── kyverno
│       ├── disallow-latest-tag.yaml
│       ├── disallow-privileged-containers.yaml
│       ├── require-resource-limits.yaml
│       └── require-standard-labels.yaml
├── README.md
└── values
    └── .gitkeep
```

Two new Applications in `apps/` and four ClusterPolicies in `policies/kyverno/`. Helm values stay inlined in the Application manifest for now; a separate `values/` directory makes sense only once configs grow large enough to warrant it.

---

### Wave 1: The Kyverno Application

Kyverno gets sync wave 1, which means ArgoCD deploys it after Cilium (wave 0) is confirmed healthy and before the policies (wave 2). Here is the full Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kyverno
  namespace: argocd
  annotations:
    argocd.argoproj.io/compare-options: ServerSideDiff=true,IncludeMutationWebhook=true
    argocd.argoproj.io/sync-wave: "1"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://kyverno.github.io/kyverno/
    chart: kyverno
    targetRevision: 3.7.1

    helm:
      values: |
        admissionController:
          replicas: 1
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi

        backgroundController:
          replicas: 1
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 256Mi

        cleanupController:
          replicas: 1
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 256Mi

        reportsController:
          replicas: 1
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 256Mi

        policyReportsCleanup:
          enabled: true

        serviceMonitor:
          enabled: false

        config:
          preserve: false
        webhookLabels:
          app.kubernetes.io/managed-by: argocd

  destination:
    server: https://kubernetes.default.svc
    namespace: kyverno

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

A few decisions worth calling out:

- **`compare-options: ServerSideDiff=true,IncludeMutationWebhook=true`**: This annotation is required for Kyverno specifically. Kyverno's admission webhook mutates objects as they pass through, so ArgoCD sees the post-mutation state as different from what is in Git and marks the application `OutOfSync` on every reconcile. Setting `ServerSideDiff=true` tells ArgoCD to use the Kubernetes API server's server-side apply dry-run to compute diffs, which accounts for mutations. Without this, the Kyverno Application will perpetually show as OutOfSync even when everything is correct.
- **`sync-wave: "1"`**: Kyverno cannot admit resources until it is running. The wave ordering ensures ArgoCD waits for Cilium (wave 0) to be healthy before starting Kyverno, and waits for Kyverno (wave 1) to be healthy before applying policies (wave 2).
- **`replicas: 1` on all four controllers**: Kyverno defaults to 3 replicas on each of its four controllers: admission, background, cleanup, and reports. On a single-node cluster, the default pod anti-affinity rules prevent more than one replica from scheduling per node. The result is three Pending pods per controller. Setting replicas to 1 fixes this. If you are running a multi-node cluster, remove these overrides.
- **`ServerSideApply=true`**: Required for Kyverno CRDs, same as Cilium. Kyverno uses server-side apply to manage field ownership on its custom resources.
- **`policyReportsCleanup.enabled: true`**: Kyverno generates `PolicyReport` resources for every policy evaluation. Without cleanup enabled, these accumulate indefinitely. Enabling cleanup keeps the cluster tidy.
- **`serviceMonitor.enabled: false`**: I will flip this to `true` when I add Prometheus. No point creating a ServiceMonitor with nothing to scrape it.

---

### Wave 2: The Kyverno Policies Application

The policies Application is simpler, it points directly to `policies/kyverno/` in this same repository:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kyverno-policies
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://gitlab.com/pras-labs/kube-gitops-demo.git
    targetRevision: main
    path: policies/kyverno/

  destination:
    server: https://kubernetes.default.svc
    namespace: kyverno

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true

  ignoreDifferences:
    - group: kyverno.io
      kind: ClusterPolicy
      jqPathExpressions:
        - ".spec.rules[]?.skipBackgroundRequests"
        - ".spec.rules[]?.validate.allowExistingViolations"
      jsonPointers:
        - /status
```

The `ignoreDifferences` block is doing important work here:

- **`/status`**: Kyverno writes status information (ready conditions, rule counts) to `ClusterPolicy.status` at runtime. ArgoCD does not own this field, but without `ignoreDifferences`, it would see the status as drift and try to revert it on every sync, fighting Kyverno in a loop.
- **`skipBackgroundRequests` and `allowExistingViolations`**: Kyverno injects these fields into policy rules automatically if they are not set. If they are missing from the Git-committed YAML, ArgoCD sees them as additions and marks the policy OutOfSync after the first apply. Ignoring these jq paths tells ArgoCD to leave Kyverno's auto-injected fields alone.

The `targetRevision: main` means this Application tracks the `main` branch. Every `git push` to `main` that changes a policy file will trigger ArgoCD to reconcile automatically.

---

### The Four Baseline Policies

All four policies start with `validationFailureAction: Audit`. In Audit mode, policy violations are recorded in `PolicyReport` resources but do not block workload creation. This gives you visibility into what is non-compliant without risking a broken deploy. The lifecycle is: ship in Audit, read the PolicyReports, verify no legitimate workloads are flagged, then flip to Enforce in a follow-up commit.

#### Disallow Latest Tag

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
  annotations:
    policies.kyverno.io/title: Disallow Latest Tag
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/severity: medium
    policies.kyverno.io/description: >-
      Container images must use a specific tag (not 'latest' or untagged).
      Use immutable tags or image digests for production workloads.
spec:
  validationFailureAction: Audit
  background: true
  rules:
    - name: check-image-tag
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Image tag 'latest' (or no tag) is not allowed. Pin images to a specific tag or digest."
        foreach:
          - list: "request.object.spec.containers"
            deny:
              conditions:
                any:
                  - key: "{{ element.image }}"
                    operator: Equals
                    value: "*:latest"
                  - key: "{{ element.image }}"
                    operator: AnyNotIn
                    value:
                      - ":"
```

This policy uses `foreach` to iterate over every container in the pod spec and applies two conditions:

- **`Equals *:latest`**: Matches any image with a `:latest` tag explicitly.
- **`AnyNotIn ":"`**: Matches images with no `:` character at all, i.e., images with no tag specified, which Docker resolves to `latest` by default.

The `latest` tag is mutable. The same tag can point to a different digest each time you pull, which makes rollbacks unreliable and deployments non-deterministic. Pinning to a specific semver tag or digest is a basic hygiene requirement for anything you care about reproducibility on.

#### Disallow Privileged Containers

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged-containers
  annotations:
    policies.kyverno.io/title: Disallow Privileged Containers
    policies.kyverno.io/category: Pod Security
    policies.kyverno.io/severity: high
    policies.kyverno.io/description: >-
      Privileged containers have unrestricted access to the host. This policy
      disallows setting securityContext.privileged=true.
spec:
  validationFailureAction: Audit
  background: true
  rules:
    - name: check-privileged
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Privileged containers are not allowed."
        pattern:
          spec:
            =(initContainers):
              - =(securityContext):
                  =(privileged): "false"
            containers:
              - =(securityContext):
                  =(privileged): "false"
```

Kyverno's `=()` syntax means "if this field exists, it must match the value". A pod that does not set `securityContext.privileged` at all passes, the policy only acts when the field is present and set to `true`. This avoids flagging the large majority of workloads that simply do not set the field.

I am starting this in Audit mode specifically because of what is coming next: Falco (Phase 2c) requires a privileged container to load its kernel module. When I add Falco I will create a `PolicyException` for it and then flip this policy to Enforce. Shipping in Audit first lets me verify that only Falco generates violations before I gate on it.

#### Require Resource Limits

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
  annotations:
    policies.kyverno.io/title: Require Resource Limits
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/severity: medium
    policies.kyverno.io/description: >-
      All containers must define CPU and memory limits.
spec:
  validationFailureAction: Audit
  background: true
  rules:
    - name: check-container-limits
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "CPU and memory limits are required for all containers."
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    cpu: "?*"
                    memory: "?*"
```

`?*` in Kyverno pattern matching means "must exist and be non-empty". This ensures every container declares both a CPU limit and a memory limit. Containers without limits can consume unbounded node resources and starve adjacent workloads; the noisy-neighbor problem. Kubernetes schedulers also make better placement decisions when limits are set.

#### Require Standard Labels

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-standard-labels
  annotations:
    policies.kyverno.io/title: Require Standard Labels
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/severity: low
    policies.kyverno.io/description: >-
      Pods must carry app.kubernetes.io/name and app.kubernetes.io/version labels
      for consistent observability and tooling support.
spec:
  validationFailureAction: Audit
  background: true
  rules:
    - name: check-standard-labels
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Pods must have 'app.kubernetes.io/name' and 'app.kubernetes.io/version' labels."
        pattern:
          metadata:
            labels:
              app.kubernetes.io/name: "?*"
              app.kubernetes.io/version: "?*"
```

The `app.kubernetes.io/name` and `app.kubernetes.io/version` labels are part of the Kubernetes recommended label set. Tools like Helm, ArgoCD, and Hubble use them for service discovery, filtering, and display. Without them, dashboards and observability tooling end up with unlabeled workloads that are hard to correlate. This policy is severity `low` because it is a convention enforcement, not a security control, but it catches label hygiene problems early before they pile up.

---

### Reading Policy Violations

With all four policies in Audit mode, Kyverno writes violations to `PolicyReport` resources. You can check what is flagged without touching any live workloads:

```bash
kubectl get policyreport -A
kubectl get clusterpolicyreport -A
```

For a detailed view of a specific report:

```bash
kubectl get policyreport -n <namespace> <report-name> -o yaml
```

The report shows each policy rule, which pod triggered it, the failure message, and whether the result was `pass` or `fail`. In Audit mode these are informational. The workflow I follow is: let the lab run for a day or two, review the reports, address any legitimate violations in the workload manifests, and only then flip `validationFailureAction` from `Audit` to `Enforce` in a Git commit.

---

## Conclusion

Phase 2b adds an admission control layer to the Kube GitOps Lab:

- **Kyverno**: Admission controller with four sub-components, all tuned to `replicas: 1` for single-node k3s
- **ArgoCD sync waves**: Wave 1 for Kyverno, wave 2 for policies, ordering guarantees the controller is healthy before its policies are admitted
- **ClusterPolicy**: Four baseline policies covering image tags, privileged containers, resource limits, and standard labels
- **PolicyReport**: Audit-mode violations captured without blocking workloads

The key practical lesson from this phase: `ignoreDifferences` on `ClusterPolicy.status` and Kyverno's auto-injected fields is not optional. Skip it and ArgoCD fights Kyverno's own runtime writes in a sync loop that never resolves. The `ServerSideDiff` compare option on the Kyverno Application has the same story; without it, mutation webhooks appear as drift.

---

## What's Next

Phase 2c will add Falco and FalcoSidekick for runtime security. Falco needs a privileged container to attach its kernel module, which means I will need a `PolicyException` to exempt it from the `disallow-privileged-containers` policy before flipping that one to Enforce. That is a good preview of the policy lifecycle in practice: Audit reveals the exception needed, write the exception, then Enforce.

Stay tuned for next stories from Pras' Labs: The Hands-On Engineer!

---

_Kubernetes, GitOps, ArgoCD, Kyverno, DevOps, Platform Engineering, k3s, Admission Control_
