# Kube GitOps Lab 03: Adding Runtime Security with Falco and FalcoSidekick

In the [previous post](https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-2b-initial-kyverno-policies) I added Kyverno as an admission controller and deployed four baseline ClusterPolicies via ArgoCD. One of those policies, `disallow-privileged-containers`, I intentionally shipped in Audit mode with a note that Falco was coming and would need a `PolicyException` before the policy could move to Enforce. This post is where that promise gets delivered.

Phase 2c adds Falco and FalcoSidekick via ArgoCD: a `PolicyException` to unblock Falco's privileged DaemonSet, then the Falco application itself (Wave 3), and finally FalcoSidekick with its web UI (Wave 4). By the end of this post, the cluster has both admission-time enforcement and runtime syscall monitoring running under the same GitOps workflow.

The code for this phase is here: [https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-2c-setup-falco-falcosidekick](https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-2c-setup-falco-falcosidekick)

---

### Admission Control vs. Runtime Security

Kyverno and Falco are not competing tools, they operate at different points in the pod lifecycle and catch different classes of problems.

Kyverno is the last gate before a workload reaches the scheduler. It blocks misconfigurations at creation time: wrong image tag, missing resource limits, privileged container. If a pod spec violates a Kyverno policy in Enforce mode, the API server rejects it and the pod never runs. This is cheap to enforce and stops obvious issues early.

Falco operates after the pod is already running. It attaches to the Linux kernel using eBPF and monitors syscalls in real time. It catches behavior that Kyverno has no visibility into: a container reading `/etc/shadow`, a process spawning a shell inside a running pod, a binary executing from `/tmp`. These are not misconfigurations in the manifest, they are anomalous behaviors at runtime that only show up after the workload is live.

Both layers are worth running. Kyverno reduces the attack surface at deploy time. Falco catches what slips through or what an attacker does after getting in.

---

### The PolicyException: Unblocking Falco Before It Lands

Falco's modern eBPF driver needs to attach probes to kernel tracepoints. On most Linux distributions this requires elevated privileges. Without an exception, the `disallow-privileged-containers` Kyverno policy would block Falco's DaemonSet pods from scheduling.

This is the policy lifecycle I described in Lab 02: ship in Audit mode, identify which legitimate workloads generate violations, write a scoped `PolicyException`, then flip to Enforce. Here is the exception:

```yaml
apiVersion: kyverno.io/v2beta1
kind: PolicyException
metadata:
  name: allow-privileged-falco
  namespace: falco-system
spec:
  exceptions:
    - policyName: disallow-privileged-containers
      ruleNames:
        - check-privileged
  match:
    any:
      - resources:
          kinds:
            - Pod
          namespaces:
            - falco-system
          selector:
            matchLabels:
              app.kubernetes.io/name: falco
```

A few things worth noting about how this exception is scoped:

- **`namespace: falco-system`**: The exception only applies inside the `falco-system` namespace. A privileged pod in any other namespace is still blocked.
- **`matchLabels: app.kubernetes.io/name: falco`**: Further narrowed to pods with the Falco Helm chart's label. This means FalcoSidekick (which is in the same namespace but does not need privileges) is not covered by this exception.
- **`ruleNames: [check-privileged]`**: Only the `check-privileged` rule is excepted. Other rules in the same policy still apply to Falco pods.

With this exception committed to the `policies/kyverno/policy-exceptions/` path, ArgoCD picks it up via the Wave 2 `kyverno-policies` Application. Falco's pods can schedule. The `disallow-privileged-containers` policy stays in Audit for now, once Falco is confirmed running cleanly I will flip it to Enforce in a follow-up commit.

---

### The Repository Structure After Phase 2c

```bash
kube-gitops-demo
├── apps
│   ├── 00-cilium.yaml
│   ├── 01-kyverno.yaml
│   ├── 02-kyverno-policies.yaml
│   ├── 03-falco.yaml
│   └── 04-falcosidekick.yaml
├── bootstrap
│   ├── argocd-apps.yaml
│   └── argocd-install.yaml
├── policies
│   └── kyverno
│       ├── policy-exceptions
│       │   └── allow-privileged-falco.yaml
│       ├── disallow-latest-tag.yaml
│       ├── disallow-privileged-containers.yaml
│       ├── require-resource-limits.yaml
│       └── require-standard-labels.yaml
├── values
│   └── .gitkeep
└── README.md
```

Two new Application manifests in `apps/`, and the `policies/kyverno/` tree now has a `policy-exceptions/` subdirectory. Because the `kyverno-policies` Application points at `policies/kyverno/` recursively, the new exception file gets picked up automatically on the next sync, no changes to the Application itself needed.

---

### Wave 3: The Falco Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: falco
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "3"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://falcosecurity.github.io/charts
    chart: falco
    targetRevision: 8.0.1

    helm:
      values: |
        driver:
          kind: modern_ebpf

        collectors:
          containerd:
            enabled: true
            socket: /run/k3s/containerd/containerd.sock

        falco:
          json_output: true

          grpc:
            enabled: true
          grpc_output:
            enabled: true

          http_output:
            enabled: true
            url: "http://falcosidekick.falco-system.svc.cluster.local:2801"

        tty: true

        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi

        tolerations:
          - operator: Exists

        serviceMonitor:
          enabled: false

  destination:
    server: https://kubernetes.default.svc
    namespace: falco-system

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

Each non-obvious setting has a specific reason:

- **`sync-wave: "3"`**: Falco deploys after Kyverno policies (Wave 2) are applied. This matters because the `PolicyException` lives in the `kyverno-policies` Application. If Falco tries to schedule before the exception exists, the DaemonSet pods get blocked.
- **`driver.kind: modern_ebpf`**: No kernel headers or kernel module compilation needed. Modern eBPF works on kernels ≥ 5.8, and k3s ships with a kernel new enough to support it. The older `ebpf` driver and the `kmod` driver both require matching kernel headers on the host, which adds host-level setup overhead that I want to avoid in this lab.
- **`collectors.containerd.socket: /run/k3s/containerd/containerd.sock`**: k3s puts its containerd socket at a non-standard path. Without this override, Falco's container collector looks at the default `/run/containerd/containerd.sock` path, finds nothing, and loses the ability to enrich Falco events with container metadata (pod name, namespace, image). Container metadata is what makes Falco alerts actionable, without it you get raw syscall events with no context about which workload triggered them.
- **`json_output: true`**: Structured JSON output, required for FalcoSidekick to parse and route events reliably.
- **`grpc.enabled: true` + `grpc_output.enabled: true`**: FalcoSidekick can consume Falco events via gRPC as an alternative transport. Enabling both HTTP and gRPC gives FalcoSidekick flexibility in how it connects.
- **`http_output.url`**: Points at the FalcoSidekick Service in the same namespace. FalcoSidekick listens on port 2801 for incoming Falco events.
- **`tty: true`**: On single-node k3s, without a pseudo-TTY, Falco's output can get buffered and log lines appear delayed or missing. Setting `tty: true` forces line-buffered output and makes logs appear immediately.
- **`tolerations: [{operator: Exists}]`**: A DaemonSet that cannot tolerate all taints will skip tainted nodes. On a single-node k3s cluster this means the DaemonSet pod never schedules if the node has any taint. `operator: Exists` matches all taints regardless of key or value.
- **`serviceMonitor.enabled: false`**: Prometheus is not in the cluster yet. I will flip this when it arrives.

---

### Wave 4: FalcoSidekick

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: falcosidekick
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "4"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://falcosecurity.github.io/charts
    chart: falcosidekick
    targetRevision: 0.13.1

    helm:
      values: |
        replicaCount: 1

        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi

        webui:
          enabled: true
          replicaCount: 1
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi

        serviceMonitor:
          enabled: false

        # slack:
        #   webhookurl: ""

  destination:
    server: https://kubernetes.default.svc
    namespace: falco-system

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

Key decisions here:

- **`sync-wave: "4"`**: FalcoSidekick should start after the Falco DaemonSet is running. If FalcoSidekick comes up first, there is no impact, it just idles until Falco starts sending events, but the ordering reflects the logical dependency.
- **`replicaCount: 1`**: FalcoSidekick's Helm chart defaults to 2 replicas with pod anti-affinity. On a single-node cluster the second replica gets stuck in Pending because both pods cannot be placed on the same node. Setting `replicaCount: 1` (and the same for `webui.replicaCount`) fixes this. Remove the override on a multi-node cluster.
- **`webui.enabled: true`**: The bundled web UI provides a live dashboard of Falco alerts. For a lab this is the fastest way to verify that the Falco -> FalcoSidekick pipeline is working, you can watch events flow in real time without tailing logs.
- **Slack config commented out**: The Slack webhook configuration is scaffolded but not active. When you are ready to wire up alerting, uncomment and fill in the `webhookurl`. FalcoSidekick supports over 50 output targets including Slack, PagerDuty, webhooks, S3, and Elasticsearch.

---

### Verifying the Setup

Once ArgoCD syncs Waves 3 and 4, verify the DaemonSet and deployment came up:

```bash
# Falco DaemonSet
kubectl get ds -n falco-system
kubectl logs -n falco-system -l app.kubernetes.io/name=falco --tail=20

# FalcoSidekick
kubectl get deploy -n falco-system
```

Port-forward the FalcoSidekick UI to see the live alert dashboard:

```bash
kubectl port-forward -n falco-system svc/falcosidekick-ui 2802:2802
```

Open `http://localhost:2802` in a browser. You should see a dashboard with recent Falco events. If Falco is running but you want to trigger a test event, launch a privileged pod:

```bash
kubectl run test-privileged \
  --image=alpine \
  --overrides='{"spec":{"containers":[{"name":"test","image":"alpine","securityContext":{"privileged":true}}]}}'
```

Falco will detect the privileged container launch and FalcoSidekick's UI will show the alert within seconds. Clean up after:

```bash
kubectl delete pod test-privileged
```

---

## Conclusion

Phase 2c adds runtime security to the Kube GitOps Lab:

- **PolicyException**: Scoped to `falco-system` namespace and `app.kubernetes.io/name: falco` pods only. Exempts the `check-privileged` rule so Falco's DaemonSet can schedule. This is the Audit -> Exception -> Enforce lifecycle from Lab 02 playing out in practice.
- **Falco (Wave 3)**: eBPF-based kernel monitoring with three k3s-specific overrides: modern eBPF driver, non-standard containerd socket path, and `tty: true` for reliable log output. HTTP and gRPC output both wired to FalcoSidekick.
- **FalcoSidekick (Wave 4)**: Alert router with live web UI. Single replica to avoid anti-affinity issues on a single-node cluster. Slack and other outputs are scaffolded and ready to enable.

Admission control and runtime security are complementary, not competing. Kyverno stops misconfigurations before the pod runs. Falco catches anomalous behavior after it is running. Running both gives you defense in depth without duplicate effort: each tool covers the window the other cannot see.

---

## What's Next

Phase 2d will add Trivy Operator and Kubescape for image vulnerability scanning and security posture assessment. With Falco in place for runtime and Kyverno for admission, the next gap is visibility into what known vulnerabilities are already running in the cluster. Trivy Operator handles that continuously at the workload level, and Kubescape covers cluster-wide CIS benchmark posture.

Stay tuned for next stories from Pras' Labs: The Hands-On Engineer!

---

_Kubernetes, GitOps, ArgoCD, Falco, FalcoSidekick, DevOps, Platform Engineering, k3s, Runtime Security_
