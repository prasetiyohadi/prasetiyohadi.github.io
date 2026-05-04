# Kube GitOps Lab 04: Trivy Operator + Kubescape for Scanning and Posture Management in GitOps

In the [previous post](https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-2c-setup-falco-falcosidekick) I added Falco and FalcoSidekick via ArgoCD, giving the cluster real-time syscall monitoring alongside the admission-time enforcement from Kyverno. Phase 2c left one gap: we can block misconfigurations and detect runtime anomalies, but we have no automated inventory of which container images carry known CVEs, and no continuous check of whether the cluster's configuration drifts from established security frameworks.

Phase 2d fills that gap. This post walks through deploying Trivy Operator (Wave 5) for vulnerability and configuration scanning, and Kubescape (Wave 6) for posture assessment against NSA, MITRE, and CIS benchmarks. I also ran into a real problem on first deploy, a disk I/O thundering herd that made the single-node k3s cluster nearly unresponsive, and the config tuning that fixed it.

The code for this phase is here: [https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-2d-setup-trivy-operator-and-kubescape](https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-2d-setup-trivy-operator-and-kubescape)

---

### Where the Security Stack Stands

After Phase 2c, the cluster has three layers of defense:

- **Wave 0: Cilium**: CNI with eBPF-based networking and Hubble observability
- **Wave 1: Kyverno**: Admission controller blocking bad manifests before pods run
- **Wave 2: Kyverno Policies**: Four ClusterPolicies (no latest tag, no privileged containers, resource limits required, standard labels required)
- **Wave 3: Falco**: Runtime syscall monitoring via eBPF
- **Wave 4: FalcoSidekick**: Alert routing and web UI for Falco events

What's missing is retrospective scanning. Kyverno and Falco operate against running or incoming workloads. They don't answer questions like: which container images have unpatched CVEs right now? Does this cluster's RBAC configuration comply with the CIS Kubernetes Benchmark? Trivy Operator and Kubescape answer those questions, continuously.

---

### Wave 5: Trivy Operator

Trivy Operator is an in-cluster controller that continuously reconciles Kubernetes resources and emits scan results as native CRDs. It runs `VulnerabilityReport`, `ConfigAuditReport`, and `RbacAssessmentReport` objects for every workload in the cluster. The scan results live in Kubernetes, which means they can be queried with `kubectl`, watched by policy engines, and shipped to dashboards without exporting to an external system.

The default Trivy Operator configuration is tuned for multi-node clusters with plenty of headroom. On a single-node k3s lab it creates problems, which I will get to in a moment. Here is the ArgoCD Application with the adjustments I made:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: trivy-operator
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "5"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://aquasecurity.github.io/helm-charts/
    chart: trivy-operator
    targetRevision: 0.32.1

    helm:
      values: |
        excludeNamespaces: "kube-system,cilium,argocd,falco-system,trivy-system,kubescape"

        operator:
          scanJobTimeout: 10m
          vulnerabilityScannerEnabled: true
          configAuditScannerEnabled: true
          rbacAssessmentScannerEnabled: true
          replicas: 1
          scanJobsConcurrentLimit: 2
          scanNodeCollectorLimit: 1
          scanJobsRetryDelay: 60s
          vulnerabilityScannerScanOnlyCurrentRevisions: true
          configAuditScannerScanOnlyCurrentRevisions: true
          builtInTrivyServer: true

        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi

        serviceMonitor:
          enabled: false

  destination:
    server: https://kubernetes.default.svc
    namespace: trivy-system

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

The settings that matter most for a single-node lab:

- **`builtInTrivyServer: true`**: This switches Trivy Operator from Standalone mode to Server mode. In Standalone mode (the default), every scan Job pod has an init container that independently downloads the full Trivy vulnerability database, roughly 500 MB compressed. In Server mode, the operator downloads the database once and serves it to all scan jobs internally. On a single node this is the most important change: it cuts DB download I/O from N-concurrent-jobs × 500 MB to one download total.
- **`scanJobsConcurrentLimit: 2`**: Reduces the default of 10 concurrent scan jobs to 2. On first deploy Trivy Operator discovers every workload in the cluster and queues them all immediately. Capping at 2 concurrent jobs means the scanner works through the queue steadily instead of spiking.
- **`scanNodeCollectorLimit: 1`**: Limits node-level collector concurrency to 1.
- **`scanJobsRetryDelay: 60s`**: Adds a 60-second backoff between retries so a failing scan doesn't tightly loop.
- **`vulnerabilityScannerScanOnlyCurrentRevisions: true`** and **`configAuditScannerScanOnlyCurrentRevisions: true`**: Skips rescanning older ReplicaSet revisions that are no longer running. Reduces scan volume significantly in clusters with deployment history.
- **`excludeNamespaces`**: Skips scanning system namespaces. Cilium, ArgoCD, Falco, and Trivy's own namespace are excluded. This avoids redundant scans of infrastructure components and reduces total scan volume.

---

### Wave 6: Kubescape

Kubescape is a security posture management tool. Where Trivy Operator looks at what is in your container images, Kubescape looks at how your cluster is configured: RBAC policies; pod security context; network policies; API server flags, and scores them against frameworks like NSA-CISA Kubernetes Hardening Guidance, MITRE ATT&CK, and CIS Kubernetes Benchmark.

Here is the ArgoCD Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kubescape
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "6"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://kubescape.github.io/helm-charts/
    chart: kubescape-operator
    targetRevision: 1.30.6

    helm:
      values: |
        clusterName: management

        continuousScan:
          enabled: true

        vulnerabilityScan:
          enabled: true

        configurationScan:
          enabled: true

        capabilities:
          runtimeObservability: disable
          networkPolicyService: disable

        kubevulnScheduler:
          scanSchedule: "0 */6 * * *"
        kubescapeScheduler:
          scanSchedule: "0 */6 * * *"

        resources:
          requests:
            cpu: 200m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi

        serviceMonitor:
          enabled: false

  destination:
    server: https://kubernetes.default.svc
    namespace: kubescape

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

The key decisions here:

- **`clusterName: management`**: Required by Kubescape to label scan results. Without it the operator deploys but results are harder to correlate.
- **`continuousScan`, `vulnerabilityScan`, `configurationScan`**: All three enabled. `continuousScan` watches for resource changes and triggers targeted rescans. `vulnerabilityScan` runs the `kubevuln` component, which scans image layers against the Kubescape vulnerability database. `configurationScan` runs posture checks against the hardening frameworks.
- **`runtimeObservability: disable`**: This disables the `node-agent` DaemonSet that Kubescape deploys for eBPF-based runtime observability. On a single-node lab the node-agent adds substantial I/O and CPU overhead on top of Falco, which is already covering the runtime layer. Disabling it keeps Kubescape focused on what it does better than Falco: posture and vulnerability scoring.
- **`networkPolicyService: disable`**: Disables Kubescape's automatic network policy generation. On a single-node lab this feature does not add value and saves resources.
- **`scanSchedule: "0 */6 * * *"`**: Stagger scans to run every 6 hours instead of the continuous polling default. Both `kubevulnScheduler` and `kubescapeScheduler` use the same schedule. This gives the node breathing room between full-cluster sweeps.
- **Resource requests raised to 200m CPU / 512 MiB memory**: Kubescape's own documentation recommends at least 500m CPU and 500 MiB memory. The default chart values are lower. Under-provisioned, the operator thrashes, it OOMs or CPU throttles during the initial scan and enters a crash loop.

---

### The Thundering Herd Problem

I want to be honest about what happened on first deploy before I landed on the config above.

I pushed Phase 2d with both operators at their chart defaults and ran `argocd app sync`. Within about two minutes the single-node k3s cluster became nearly unresponsive. Disk I/O spiked to 100% utilization and stayed there for several minutes. ArgoCD's UI was sluggish. SSH sessions lagged. The node recovered eventually, but it was not a clean deployment.

After digging in, I identified three compounding causes:

**Cause 1: Trivy Operator's initial scan storm.** The default `scanJobsConcurrentLimit` is 10. On first deployment the operator discovers every workload in the cluster simultaneously and spawns up to 10 concurrent scan Jobs. In Standalone mode (the default), each scan Job pod has an init container that independently downloads and extracts the full Trivy DB. On a single node: 10 concurrent pods × 500 MB DB download + extraction = massive read/write spike. Scan results are then written back to etcd as CRD objects, adding more I/O.

**Cause 2: Kubescape's initial full cluster scan.** When `continuousScan: enabled` with default settings, Kubescape immediately triggers a full cluster scan on first deployment. The `kubevuln` component downloads its own vulnerability database and begins scanning all container image layers. The `node-agent` DaemonSet (enabled by default) runs eBPF probes alongside Falco, compounding CPU and I/O pressure.

**Cause 3: Wave timing.** ArgoCD deploys Wave 5 and Wave 6 nearly simultaneously. There is a slight ordering guarantee (Wave 5 resources must reach a healthy state before Wave 6 starts), but in practice both operators were actively scanning within a few minutes of each other. The thundering herd was not one tool but two operating at peak I/O simultaneously.

The fixes I applied address each cause directly:

- `builtInTrivyServer: true` -> Trivy DB downloaded once, not per job
- `scanJobsConcurrentLimit: 2` -> Trivy scan queue drains slowly instead of spiking
- `runtimeObservability: disable` -> Kubescape's node-agent stays off, Falco covers runtime
- `scanSchedule: "0 */6 * * *"` -> Kubescape scans on a schedule instead of immediately looping
- Raised resource requests to match Kubescape's documented minimums -> no OOM thrashing

After those changes, a fresh deploy stays responsive throughout.

---

### Version Bumps

Phase 2d also pinned previously floating chart versions across the stack:

- **Cilium** `1.19.2` (previously `1.19.x`)
- **Kyverno** `3.7.1` (previously `3.3.4`)
- **Falco** `8.0.1` (previously `4.x`)
- **FalcoSidekick** `0.13.1` (previously `0.8.x`)

Pinning versions is a GitOps discipline: the Git history becomes an exact audit log of what version was running and when, not a floating reference that silently changes on the next sync. I updated the comments in each Application manifest to reflect the exact version rather than a range.

---

### Verifying the Deployment

Once both operators sync cleanly, verify scan results are flowing:

```bash
# Check VulnerabilityReports across all namespaces
kubectl get vulnerabilityreports -A

# Check ConfigAuditReports
kubectl get configauditreports -A

# Check RbacAssessmentReports
kubectl get rbacassessmentreports -A

# Check Kubescape scan results
kubectl get workloadconfigurationscans -A
kubectl get vulnerabilitymanifests -A
```

On a fresh cluster it takes several minutes for the first round of scans to complete, especially while Trivy Operator works through the queue at the throttled concurrency limit. The `VulnerabilityReport` objects appear namespace by namespace as scans finish.

A quick sanity check on the Trivy Operator server mode, you should see only one DB download rather than one per scan job:

```bash
kubectl logs -n trivy-system deployment/trivy-operator | grep "downloading"
```

---

### What Was Built

Phase 2d adds the final two security layers to the single-node GitOps cluster:

- **Trivy Operator** scans every workload's container images for CVEs, checks resource configurations for misconfigurations, and assesses RBAC bindings; all as Kubernetes-native CRD objects, continuously reconciled
- **Kubescape** scores the cluster's overall security posture against NSA-CISA, MITRE ATT&CK, and CIS Kubernetes Benchmark frameworks, with scheduled full-cluster scans every 6 hours

Together with Kyverno (admission control) and Falco (runtime detection), the cluster now has:

1. **Admission-time enforcement:** bad manifests are blocked before pods run
2. **Runtime detection:** anomalous syscall behavior is flagged in real time
3. **Image scanning:** CVEs in container images are surfaced as queryable Kubernetes objects
4. **Posture assessment:** cluster configuration is continuously scored against hardening frameworks

The practical takeaway from this phase: both Trivy Operator and Kubescape are well-designed tools that assume more resources than a single-node lab provides. The default configurations are not wrong, they are tuned for production multi-node clusters. On constrained infrastructure, the thundering herd behavior is real, and the tuning parameters exist specifically to address it. Read the documentation on `builtInTrivyServer` and Kubescape's capability flags before deploying to a resource-constrained environment.

---

### What's Next

Phase 3 is multi-cluster provisioning with Crossplane; managing external infrastructure (cloud resources, additional clusters) through the same GitOps workflow that manages the security stack. The single-node lab will extend into a management cluster pattern where ArgoCD and Crossplane together drive infrastructure lifecycle without any manual cloud console interaction.

Stay tuned for next stories with Pras' Labs: The Hands-On Engineer!

---

_Tags: Kubernetes, GitOps, DevSecOps, Trivy, Kubescape, ArgoCD, Security, k3s, Platform Engineering_
