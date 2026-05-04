# Social Posts: Kube GitOps Lab 01 — Bootstrapping Cilium and ArgoCD on a Single-Node k3s Cluster

---

## 𝕏 Twitter/X

**Post 1** (hook):
```
Tried to let ArgoCD deploy Cilium. Problem: ArgoCD needs pod networking to run. Cilium provides pod networking. Classic deadlock.

Solution: install Cilium via Helm first, then ArgoCD adopts it. One manual step, then GitOps takes over forever.
```
*(277 chars)*

**Post 2** (CTA):
```
New post: Kube GitOps Lab #1 — bootstrapping Cilium + ArgoCD on k3s.

Covers the CNI deadlock, adopt-then-manage pattern, sync waves, and why you need ignoreDifferences on cilium-config.

#Kubernetes #GitOps #Cilium #ArgoCD #DevOps #HandsOnEngineer
```
*(256 chars)*

---

## 💼 LinkedIn

```
Most GitOps guides skip the first hard problem: you can't let ArgoCD deploy your CNI.

ArgoCD needs pod networking to run. The CNI provides pod networking. If you try to GitOps your way through bootstrap, nothing starts.

I ran into this building Kube GitOps Lab — a hands-on series where I set up a production-grade GitOps stack on Kubernetes one layer at a time.

In the first post I cover:

• Installing k3s without Flannel and bootstrapping ArgoCD via Helm
• The CNI bootstrap deadlock and how the adopt-then-manage pattern solves it
• The full ArgoCD Application manifest for Cilium with sync waves, kube-proxy replacement, and Hubble observability
• Why you need `ignoreDifferences` on the `cilium-config` ConfigMap or you'll end up in a sync loop

The repo is fully public. Every config file shown is the real one — no pseudocode.

If you're building GitOps platforms or just want to see how Cilium and ArgoCD fit together from scratch, this one is worth the read.

#Kubernetes #GitOps #ArgoCD #Cilium #DevOps #PlatformEngineering #k3s
```

---

## ✉️ Email Subject Lines

1. `Why can't ArgoCD deploy your CNI — and how to fix it`
2. `I bootstrapped Cilium + ArgoCD on k3s and hit a deadlock on day one`
3. `Kube GitOps Lab: Cilium adoption pattern, sync waves, and Hubble from scratch`
4. `Kube GitOps Lab #1: Bootstrapping Cilium and ArgoCD on k3s`

---

## ☁️ Bluesky

**Post 1**:
```
ArgoCD can't deploy your CNI. The CNI needs to exist for ArgoCD to run. That's the bootstrap deadlock.

Fix: install Cilium via Helm first, then ArgoCD adopts it. One exception to the GitOps rule — then it's Git-only from there.
```
*(228 chars)*

**Post 2**:
```
New post: Kube GitOps Lab #1 — k3s + ArgoCD + Cilium.

Covers sync waves, kube-proxy replacement, Hubble metrics, and the ignoreDifferences trick for cilium-config drift. Full YAML included.

gitlab.com/pras-labs/kube-gitops-demo
```
*(235 chars)*

---

## 🧵 Threads

**Post 1**:
```
I tried to let ArgoCD manage Cilium from day one. Ran straight into a deadlock.

ArgoCD needs pod networking. Cilium provides pod networking. Neither can start first.

The fix is simple but easy to miss: install Cilium manually via Helm before ArgoCD boots, then let ArgoCD adopt the running release. One manual step, then pure GitOps.
```
*(342 chars)*

**Post 2**:
```
Kube GitOps Lab #1 covers:

• k3s without Flannel + ArgoCD bootstrap via App of Apps
• Full Cilium Helm install with kube-proxy replacement and Hubble enabled
• The ArgoCD Application manifest with sync-wave 0 and why that ordering matters
• ignoreDifferences on cilium-config — skip this and you're in a sync loop forever
```
*(327 chars)*

**Post 3**:
```
This is post #1 in the Kube GitOps Lab series — building a full GitOps security stack on Kubernetes one layer at a time. Next up: Kyverno admission control.

Full post + public repo linked in profile. Every config shown is the real one.

#Kubernetes #GitOps #Cilium #ArgoCD
```
*(275 chars)*

---

## 📘 Facebook

```
Wrote up the first post in a new hands-on series: building a GitOps stack on Kubernetes from scratch.

First lesson learned the hard way — you can't let ArgoCD deploy your CNI. The moment you try, you have a deadlock. The workaround (install Cilium manually first, then ArgoCD adopts it) is simple but took me a bit to land on cleanly.

Covers k3s, ArgoCD, Cilium, Hubble, and the full config. Public repo so you can follow along.

Have you hit the CNI bootstrap problem before? Curious what approach others use.
```
*(497 chars)*

---

*Hashtags: `#Kubernetes #GitOps #Cilium #ArgoCD #DevOps #PlatformEngineering #k3s #HandsOnEngineer`*
