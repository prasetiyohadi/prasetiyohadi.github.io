# Social Posts: Kube GitOps Lab 02 — Kyverno Admission Control via ArgoCD

---

## 𝕏 Twitter/X

**Post 1** (hook):
```
ArgoCD + Kyverno = instant OutOfSync loop — unless you set ServerSideDiff=true on the Application.

Kyverno's webhook mutates objects. ArgoCD sees drift. Sync never settles.

One annotation fixes it. Took me longer to find than I'd like to admit.
```
_(267 chars)_

**Post 2** (CTA):
```
Kube GitOps Lab 02 is live: Kyverno admission control via ArgoCD on a single-node k3s cluster.

Covers sync waves, ignoreDifferences gotchas, 4 baseline ClusterPolicies, and the Audit→Enforce lifecycle.

[link] #Kubernetes #GitOps #DevOps #Kyverno #HandsOnEngineer
```
_(272 chars)_

---

## 💼 LinkedIn

```
Most engineers add Kyverno to their cluster and immediately hit a problem: ArgoCD marks the Application OutOfSync on every reconcile, even when nothing changed.

The cause: Kyverno's admission webhook mutates objects as they pass through. ArgoCD compares what's in Git to what it sees in the cluster — and the post-mutation state looks like drift.

In the latest Kube GitOps Lab post I walk through deploying Kyverno entirely via ArgoCD on a single-node k3s cluster, including the specific annotations and config that prevent the sync loop from the start.

What's covered:

- The compare-options annotation (ServerSideDiff=true,IncludeMutationWebhook=true) that fixes the OutOfSync problem
- ArgoCD sync wave ordering: Cilium (wave 0) → Kyverno (wave 1) → policies (wave 2)
- Single-node replica workaround: Kyverno defaults to 3 replicas per controller; pod anti-affinity means two stay Pending forever
- ignoreDifferences on ClusterPolicy.status so ArgoCD stops fighting Kyverno's own runtime writes
- Four baseline ClusterPolicies (disallow latest tag, no privileged containers, require resource limits, require standard labels) — all starting in Audit mode

The Audit-first approach is deliberate. I already know Falco (next phase) needs a privileged container. Shipping the policy in Audit first lets me see exactly what's flagged before I flip to Enforce and start blocking things.

Full post + repo link in the comments.

#Kubernetes #GitOps #ArgoCD #Kyverno #DevOps #PlatformEngineering #k3s
```

---

## ✉️ Email Subject Lines

1. **Curiosity/question**: `Why does ArgoCD keep showing Kyverno as OutOfSync?`
2. **How I...**: `How I deployed Kyverno via ArgoCD without the endless sync loop`
3. **Direct/practical**: `Kyverno + ArgoCD on k3s: sync waves, ignoreDifferences, and 4 baseline policies`
4. **Series continuity**: `Kube GitOps Lab #2: Kyverno Admission Control via ArgoCD`

---

## ☁️ Bluesky

**Post 1** (hook):
```
ArgoCD + Kyverno will hit an OutOfSync loop on day one — unless you add ServerSideDiff=true to the Application annotation.

Kyverno mutates objects. ArgoCD sees drift. It never settles.

New Kube GitOps Lab post walks through it.
```
_(249 chars)_

**Post 2** (coverage + CTA):
```
Kube GitOps Lab 02: Kyverno via ArgoCD on single-node k3s.

Covers sync wave ordering, ignoreDifferences gotchas, replica=1 workaround, and 4 ClusterPolicies in Audit mode.

[link] #Kubernetes #GitOps #Kyverno
```
_(211 chars)_

---

## 🧵 Threads

**Post 1**:
```
I added Kyverno to my GitOps lab this week and immediately hit two problems:

1. ArgoCD marks the Application OutOfSync on every reconcile, even when nothing changed
2. Kyverno's own status writes cause ArgoCD to try to revert them in a loop

Both are fixable. Neither is obvious from the docs.
```

**Post 2**:
```
What the new Kube GitOps Lab post covers:

- ServerSideDiff annotation that stops the OutOfSync loop
- ignoreDifferences on ClusterPolicy.status to stop the revert loop
- Sync wave ordering: Cilium (0) → Kyverno (1) → policies (2)
- 4 baseline ClusterPolicies all starting in Audit mode so nothing breaks on day one
```

**Post 3**:
```
The Audit-first approach matters: I know Falco (next phase) needs a privileged container. I'm shipping the disallow-privileged policy in Audit so I can see what flags before I enforce it.

Full post + repo: [link]
```

---

## 📘 Facebook

```
Just published the second post in my Kube GitOps Lab series — this one on adding Kyverno admission control via ArgoCD.

The tricky part wasn't Kyverno itself, it was getting ArgoCD to stop reporting OutOfSync on every reconcile (Kyverno's mutation webhook causes it). Took a specific annotation to fix.

Also covers sync wave ordering, why all four Kyverno controllers need replicas=1 on single-node k3s, and four baseline policies in Audit mode before flipping to Enforce.

Has anyone else run into the ArgoCD + Kyverno sync loop issue? Curious how others handled it.

[link]
```

---

_Hashtags: #Kubernetes #GitOps #Kyverno #ArgoCD #DevOps #PlatformEngineering #k3s #HandsOnEngineer_
