# Kube GitOps Lab 01: Bootstrapping Cilium and ArgoCD on a Single-Node k3s Cluster

## Kube GitOps Lab Introduction

I have been running the CI/CD Lab series for a while now, working through GitLab CI pipelines, security scans, artifact signing, and more. In parallel, I wanted to build something on the Kubernetes side, a hands-on lab where I set up a full GitOps stack from scratch, document every decision, and publish the whole thing as a public repository.

This is the first post in the **Kube GitOps Lab** series. The goal is to build a production-grade GitOps platform on Kubernetes, one layer at a time: CNI, admission control, runtime security, scanning, and eventually multi-cluster provisioning. Every change goes through Git. No `kubectl apply` from a laptop once the bootstrap is done.

For this lab I use a single-node `k3s` cluster running locally. The code is in this repository: [https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-1-bootstrap-k8s-cilium-argocd-apps](https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-1-bootstrap-k8s-cilium-argocd-apps)

---

## The Stack and the Plan

Before diving in, here is the phase breakdown I settled on after some iteration. I originally had Phase 2 as one big "security stack" bucket, but that was too coarse for a lab. Breaking it into 2a/2b/2c/2d lets me ship and document each component independently.

| Phase | Scope                                           | Status     |
| ----- | ----------------------------------------------- | ---------- |
| 1     | Repo skeleton + ArgoCD on management cluster    | ✅ Done    |
| 2a    | Cilium (CNI + kube-proxy replacement + Hubble)  | ✅ Done    |
| 2b    | Kyverno (admission controller + policies)       | ⏳ Planned |
| 2c    | Falco + FalcoSidekick (runtime security)        | ⏳ Planned |
| 2d    | Trivy Operator + Kubescape (scanning + posture) | ⏳ Planned |
| 3     | Multi-cluster + Crossplane provisioning         | ⏳ Planned |

The ordering is intentional: Cilium must come first because it is the CNI. Everything else depends on pod networking being functional. This ordering constraint also creates the first interesting problem in the lab, which I will get to in a moment.

The repository structure after Phase 2a looks like this:

```bash
kube-gitops-demo
├── apps
│   └── 00-cilium.yaml
├── bootstrap
│   ├── argocd-apps.yaml
│   └── argocd-install.yaml
├── README.md
└── values
    └── .gitkeep
```

---

### Phase 1: Bootstrapping ArgoCD

Phase 1 sets up the foundation: a `k3s` cluster and ArgoCD running on it. I install `k3s` without its default Flannel CNI since I want to replace it with Cilium:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--flannel-backend=none --disable-network-policy --disable=traefik" sh -
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER ~/.kube/config
```

With `k3s` running (but pods not yet scheduled since there is no CNI), I add the ArgoCD Helm repo and install it:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --values bootstrap/argocd-install.yaml
kubectl -n argocd wait --for=condition=available deployment/argocd-server --timeout=300s
```

Then apply the root Application manifest that tells ArgoCD to watch this repository:

```bash
kubectl apply -f bootstrap/argocd-apps.yaml
```

When you run this you will see a warning:

```
Warning: metadata.finalizers: "resources-finalizer.argocd.argoproj.io": prefer
a domain-qualified finalizer name including a path (/) to avoid accidental
conflicts with other finalizer writers
```

The warning comes from the Kubernetes API server (introduced in k8s 1.20+) complaining that the finalizer name does not follow the `domain/path` format. You cannot fix this. The string `resources-finalizer.argocd.argoproj.io` is hard-coded into the ArgoCD controller, it watches for exactly that string to trigger cascade deletion when an Application is removed. If you renamed it to the preferred `argoproj.io/resources-finalizer` format, ArgoCD would ignore it and cascade deletion would silently stop working. ArgoCD is aware of this and has not changed the finalizer name for backwards compatibility. Safe to ignore.

Once ArgoCD is up, it watches the repository and will automatically discover and apply anything in the `apps/` directory. That is the App of Apps pattern: one root Application points to the repo, and all per-component Applications live in `apps/`.

---

### The CNI Bootstrap Deadlock

Here is where it gets interesting. The natural GitOps instinct is: put everything in Git, let ArgoCD deploy it. But Cilium is the CNI. ArgoCD itself needs pod networking to run, and pod networking requires Cilium. If you try to let ArgoCD deploy Cilium on first boot, you have a deadlock; ArgoCD cannot start because there is no network, and the network cannot start because ArgoCD has not deployed it yet.

The resolution is straightforward: install Cilium manually via Helm **before** ArgoCD, then let ArgoCD adopt the already-running release. From that point forward, all Cilium changes go through Git commits only. It is a one-time exception to the GitOps rule.

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
helm install cilium cilium/cilium \
  --namespace cilium \
  --create-namespace \
  --version 1.19.2 \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=127.0.0.1 \
  --set k8sServicePort=6443 \
  --set bpf.masquerade=true \
  --set ipam.mode=kubernetes \
  --set cni.exclusive=false \
  --set operator.replicas=1 \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true
```

A few flags worth calling out:

- **`kubeProxyReplacement=true`**: Cilium replaces `kube-proxy` entirely using eBPF. This is the reason I use Cilium rather than a simpler CNI, native eBPF networking without the overhead of `kube-proxy`.
- **`k8sServiceHost` / `k8sServicePort`**: Required when `kubeProxyReplacement=true`. Cilium needs to know where the API server is so it can handle service traffic directly. For single-node `k3s`, `127.0.0.1:6443` is the default.
- **`bpf.masquerade=true`**: eBPF-based masquerading instead of `iptables`. Faster and more consistent with the rest of the eBPF data plane.
- **`ipam.mode=kubernetes`**: Use Kubernetes for IP address management, simpler and correct for a single-cluster lab.
- **`cni.exclusive=false`**: Allows Cilium to coexist with other CNI configuration that may already be present.
- **`operator.replicas=1`**: Cilium operator defaults to 2 replicas. On a single-node cluster, the second pod stays `Pending` permanently because the pod anti-affinity rule prevents scheduling two operator pods on the same node. If you are running a multi-node cluster, remove this flag.
- **`hubble.enabled=true`** (and relay + UI): Hubble gives you an eBPF-based network observability layer. I want to see actual traffic flows and drop reasons from day one.

After the Helm install, pods start scheduling. Then ArgoCD comes up, discovers `apps/00-cilium.yaml`, and adopts the running Cilium release.

---

### Phase 2a: ArgoCD Adopts Cilium

With ArgoCD running and Cilium already installed, the adoption happens automatically. ArgoCD sees the Application manifest in `apps/00-cilium.yaml`, checks if a matching Helm release exists in the cluster, and takes ownership. From this point, any change to the Cilium configuration must go through a Git commit, ArgoCD enforces it.

Here is the full `apps/00-cilium.yaml`:

```yaml
# Cilium: Wave 0 (CNI Foundation)
#
# ⚠️  Bootstrap note:
# Cilium CANNOT be deployed by ArgoCD on first boot: ArgoCD itself needs pod
# networking to run, which requires Cilium. This creates a deadlock.
#
# Resolution: Cilium is installed manually via Helm BEFORE ArgoCD (see
# README.md Step 2). Once ArgoCD is running, it discovers this Application and
# ADOPTS the already-running Cilium release. From that point, all Cilium config
# changes and upgrades go through Git commits only.
#
# Chart: https://helm.cilium.io/
# Version: 1.19.x (check latest: helm search repo cilium/cilium)

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cilium
  namespace: argocd
  annotations:
    # Wave 0: deploy before everything else
    argocd.argoproj.io/sync-wave: "0"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://helm.cilium.io/
    chart: cilium
    targetRevision: 1.19.2 # pin to a specific patch; update deliberately

    helm:
      values: |
        cluster:
          name: management
          id: 1

        cni:
          exclusive: false
          chainingMode: none

        ipam:
          mode: kubernetes

        kubeProxyReplacement: true

        # k3s default API server address (single-node)
        # Update to your control-plane IP if multi-node
        k8sServiceHost: "127.0.0.1"
        k8sServicePort: "6443"

        bpf:
          masquerade: true

        hubble:
          enabled: true
          relay:
            enabled: true
          ui:
            enabled: true
          metrics:
            enabled:
              - dns:query;ignoreAAAA
              - drop
              - tcp
              - flow
              - port-distribution
              - icmp
              - http

        prometheus:
          enabled: true
          serviceMonitor:
            enabled: false

        operator:
          # Single-node lab: default replicas=2 causes second pod to stay Pending
          # (pod anti-affinity prevents scheduling two operator pods on the same node)
          replicas: 1
          prometheus:
            enabled: true
            serviceMonitor:
              enabled: false
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 256Mi

        policyEnforcementMode: default

        hostFirewall:
          enabled: false

        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi

  destination:
    server: https://kubernetes.default.svc
    namespace: cilium

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true # required for Cilium CRDs
    retry:
      limit: 5
      backoff:
        duration: 30s
        factor: 2
        maxDuration: 5m

  # ArgoCD health check: wait for DaemonSet rollout before Wave 1
  ignoreDifferences:
    - group: ""
      kind: ConfigMap
      name: cilium-config
      jsonPointers:
        - /data
```

Let me walk through the parts that are not self-explanatory:

- **`sync-wave: "0"`**: ArgoCD uses sync waves to order deployments. Wave 0 means Cilium deploys before anything in Wave 1, 2, etc. When I add Kyverno and Falco in later phases, they will use higher wave numbers so they only start after the CNI is confirmed healthy.
- **`targetRevision: 1.19.2`**: Pinned to a specific patch version. I want to control Cilium upgrades deliberately through Git commits, not have ArgoCD pull a newer chart automatically.
- **`cluster.id: 1`**: Required for Cilium's ClusterMesh feature (multi-cluster networking). Not needed now, but setting it from the start avoids a disruptive reconfiguration later when I get to Phase 3.
- **`ServerSideApply=true`**: Required for Cilium CRDs. Cilium uses server-side apply for its custom resources, and ArgoCD needs to know this to avoid field ownership conflicts.
- **`ignoreDifferences` on `cilium-config` ConfigMap**: Cilium's agent modifies the `cilium-config` ConfigMap at runtime (for example, when a node agent records node-specific configuration). Without this, ArgoCD would detect that drift and immediately try to revert it, causing a sync loop. Ignoring the `/data` path on that specific ConfigMap tells ArgoCD to let Cilium manage its own config at runtime while still enforcing everything else.
- **`prune: true` + `selfHeal: true`**: ArgoCD will remove resources that are no longer in Git and will revert any manual changes made outside of Git. This enforces GitOps discipline after the bootstrap.

After applying `bootstrap/argocd-apps.yaml`, ArgoCD will:

1. Discover `apps/00-cilium.yaml` and adopt the already-running Cilium release
2. Manage all future changes to Cilium via Git commits
3. Deploy all subsequent waves (Kyverno, Falco, etc.) automatically as they are added to `apps/`

You can verify adoption in the ArgoCD UI or via CLI:

```bash
argocd app list
argocd app get cilium
```

The application status should show `Synced` and `Healthy`. If it shows `OutOfSync`, check whether the values in `apps/00-cilium.yaml` match what was installed via Helm, any difference will cause ArgoCD to try to reconcile.

---

## Conclusion

In this post I set up the foundation for the Kube GitOps Lab:

- **k3s**: Single-node Kubernetes cluster without Flannel
- **ArgoCD**: GitOps controller using the App of Apps pattern
- **Cilium**: eBPF-based CNI with kube-proxy replacement and Hubble observability
- **Hubble**: Network traffic visibility out of the box

The key practical lesson from this phase is that the CNI bootstrap deadlock is real and you cannot GitOps your way out of it on day one. The adopt-then-manage pattern is the right answer: install the CNI manually once, let ArgoCD take ownership, and from that point treat it like any other managed resource. Every future Cilium upgrade happens via a version bump in `apps/00-cilium.yaml` — no more manual Helm commands.

---

## What's Next

In the next post, I will add Phase 2b: Kyverno as an admission controller. Kyverno does not have the same bootstrap deadlock as the CNI, so it will deploy entirely through ArgoCD from the start. I plan to set up baseline policies: requiring resource limits on all pods; blocking privileged containers; and enforcing image pull policies, and walk through how policy violations look in practice.

Stay tuned for next stories from Pras' Labs: The Hands-On Engineer!

---

_Kubernetes, GitOps, ArgoCD, Cilium, DevOps, Platform Engineering, k3s, Hubble_
