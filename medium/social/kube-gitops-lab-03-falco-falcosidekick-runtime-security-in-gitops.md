# Social Posts: Kube GitOps Lab 03 — Falco + FalcoSidekick

## 𝕏 Twitter/X

**Post 1** (278 chars)
```
Kyverno blocks privileged containers. Falco needs a privileged container to attach its eBPF probe.

I wrote a scoped PolicyException in Git, ArgoCD synced it, then deployed Falco as Wave 3.

Admission control and runtime security working together — not fighting each other.
```

**Post 2** (272 chars)
```
New Kube GitOps Lab post: Falco + FalcoSidekick on k3s via ArgoCD.

Covers the PolicyException pattern, 3 k3s-specific Falco overrides (containerd socket, tty, tolerations), and FalcoSidekick UI.

Full post: https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-2c-setup-falco-falcosidekick

#Kubernetes #GitOps #DevOps #HandsOnEngineer #RuntimeSecurity
```

---

## 💼 LinkedIn

```
Kyverno blocks privileged containers. Falco needs a privileged container to load its eBPF probe. That's the tension Phase 2c of my Kube GitOps Lab had to resolve.

The answer isn't to weaken the policy — it's to write a scoped PolicyException in Git and let ArgoCD sync it before Falco even tries to schedule.

Here's what the post covers:

• PolicyException scoped to falco-system namespace and app.kubernetes.io/name: falco pods only — FalcoSidekick in the same namespace is not covered
• Wave 3 Falco Application: modern_ebpf driver (no kernel headers), k3s containerd socket override at /run/k3s/containerd/containerd.sock, tty: true to avoid log buffering, tolerations for single-node scheduling
• Wave 4 FalcoSidekick: replicaCount: 1 to avoid anti-affinity Pending on single-node, web UI enabled for live alert dashboard, Slack output scaffolded for later
• Why admission control and runtime security aren't competing — Kyverno catches misconfigs at deploy time, Falco catches anomalous behavior after the pod is live

If you're running Falco on k3s and wondering why container metadata is missing from alerts, the containerd socket path is almost certainly the reason.

Full write-up and repo linked in comments.

#Kubernetes #GitOps #ArgoCD #Falco #RuntimeSecurity #DevOps #PlatformEngineering #k3s
```

---

## ✉️ Email Subject Lines

1. `Why does Falco need a privileged container if Kyverno already blocks them?`
2. `I added runtime security to my GitOps cluster — here's the PolicyException that made it work`
3. `Falco + FalcoSidekick on k3s via ArgoCD: 3 overrides you actually need`
4. `Kube GitOps Lab 03: Runtime Security with Falco and FalcoSidekick`

---

## ☁️ Bluesky

**Post 1** (298 chars)
```
Kyverno blocks privileged containers. Falco needs privileges to attach its eBPF probe.

I wrote a scoped PolicyException in Git — namespace-locked, label-matched — and ArgoCD synced it before Falco's Wave 3 even started.

That's the policy lifecycle: Audit → Exception → Enforce.
```

**Post 2** (294 chars)
```
New Kube GitOps Lab post covers Falco + FalcoSidekick on k3s: containerd socket path, tty buffering fix, anti-affinity replicaCount, and the FalcoSidekick web UI.

Repo + full write-up:
https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-2c-setup-falco-falcosidekick

#Kubernetes #GitOps #Falco
```

---

## 🧵 Threads

**Post 1**
```
I spent Phase 2b of my Kube GitOps Lab shipping Kyverno with disallow-privileged-containers in Audit mode — and explicitly called out that Falco was coming and would need an exception.

Phase 2c is where that promise gets delivered.

PolicyException → Falco DaemonSet (Wave 3) → FalcoSidekick (Wave 4). All through ArgoCD, all in Git.
```

**Post 2**
```
Three k3s-specific Falco gotchas the docs don't surface clearly:

• containerd socket lives at /run/k3s/containerd/containerd.sock — not the default path. Miss this and Falco runs but all events lack container metadata (pod name, namespace, image).
• tty: true — without it, log output gets buffered and appears delayed or missing on single-node k3s.
• tolerations: [{operator: Exists}] — DaemonSet won't schedule on a tainted node without it.
```

**Post 3**
```
Full post: Kube GitOps Lab 03 — Falco + FalcoSidekick on k3s

Covers the PolicyException pattern, all the Helm overrides with reasons, and verification commands including how to trigger a test Falco alert.

Repo: https://gitlab.com/pras-labs/kube-gitops-demo/-/tree/feat/phase-2c-setup-falco-falcosidekick

Next up: Phase 2d — Trivy Operator + Kubescape for image scanning and posture.
```

---

## 📘 Facebook

```
Just published Kube GitOps Lab 03 — adding Falco and FalcoSidekick for runtime security on a k3s cluster managed entirely through ArgoCD.

The interesting part: Kyverno was already blocking privileged containers, and Falco needs privileges to attach its eBPF probe. So I had to write a PolicyException first, commit it to Git, and let ArgoCD sync it before Falco could even schedule. That's the policy lifecycle in practice.

Also covers three k3s-specific quirks that catch people off guard (the containerd socket path is the big one).

Repo link in the comments if you want to follow along. Have you run Falco in a GitOps workflow before — any outputs beyond Slack worth wiring up?
```

---

Hashtags used `#Kubernetes #GitOps #RuntimeSecurity #Falco` across platforms — specific enough to reach the k8s security community without diluting into generic DevOps noise. `#HandsOnEngineer` kept on Twitter/X per series convention.
