# Social Posts: Kube GitOps Lab 05 — Crossplane Multi-Cluster Provisioning

---

## 𝕏 Twitter/X

**Post 1** (271 chars)

```
Tried to deploy Crossplane via ArgoCD in one app. It broke.

The Kubernetes provider installs its own CRDs. ProviderConfigs depend on those CRDs. Apply everything at once = race condition.

Fix: 4 separate ArgoCD apps, waves 7-10. Each wave waits for healthy before next fires.
```

**Post 2** (242 chars)

```
New post: wiring Crossplane into a GitOps setup with ArgoCD sync waves, InjectedIdentity ProviderConfigs, and an XRD + Composition that shows the abstraction pattern even without a cloud provider.

[link] #Kubernetes #GitOps #Crossplane #DevOps #HandsOnEngineer
```

---

## 💼 LinkedIn

```
Most tutorials show Crossplane working. They don't show what breaks when you try to deploy it with GitOps.

I hit a CRD race condition on my first attempt: one ArgoCD app pointing at the whole crossplane/ directory, ArgoCD applies everything at once, ProviderConfig lands before its CRD exists, sync fails.

The fix required understanding *why* it breaks — and splitting the deployment into four separate ArgoCD applications with explicit sync waves.

My latest post in the Kubernetes GitOps Lab series covers:

• The CRD race condition problem and why a single ArgoCD app for Crossplane won't work
• Wave 7–10 deployment order: Crossplane core → Providers → ProviderConfigs → XRDs/Compositions
• InjectedIdentity ProviderConfigs for local k3s (no cloud credentials needed)
• Building an XRD + Composition that demonstrates the patch mechanism honestly, even as a local lab stub

The lab runs on a single-node k3s cluster, so the Composition creates a ConfigMap instead of real cloud resources. That's not a hack — it's the same abstraction and patch wiring a real EKS/GKE Composition uses. The implementation swaps out; the API stays the same.

If you're trying to run Crossplane in a GitOps setup and hitting ordering issues, the wave split pattern is the answer.

Full post linked below.

#Kubernetes #GitOps #Crossplane #ArgoCD #PlatformEngineering #DevOps #InfrastructureAsCode
```

---

## ✉️ Email Subject Lines

1. `Why does ArgoCD + Crossplane break on first deploy?`
2. `I split one ArgoCD app into four to fix a CRD race condition`
3. `Crossplane GitOps: the correct wave ordering for providers and ProviderConfigs`
4. `Kubernetes GitOps Lab #5: Multi-Cluster Provisioning with Crossplane`

---

## ☁️ Bluesky

**Post 1** (296 chars)

```
Crossplane + ArgoCD has a gotcha: providers install their own CRDs, and ProviderConfigs depend on those CRDs. Apply everything at once = sync failure.

Solution: 4 apps, 4 waves. Each wave waits for healthy state before next fires. Wave 7 (core) → 8 (providers) → 9 (configs) → 10 (XRDs).
```

**Post 2** (289 chars)

```
New Kubernetes GitOps Lab post: Crossplane multi-cluster provisioning on k3s. Covers InjectedIdentity ProviderConfigs, XRD + Composition patch wiring, and why the wave split matters.

Local lab = ConfigMap POC instead of real clusters. The wiring is real.

[link]
```

---

## 🧵 Threads

**Post 1** (438 chars)

```
I tried to deploy Crossplane via ArgoCD the naive way — one app, one directory, sync everything.

It broke immediately. Crossplane's Kubernetes provider installs its own CRDs. ProviderConfigs live in that API group. If ArgoCD applies both at the same time, the ProviderConfig lands before its CRD exists.

The fix: split the deployment into 4 ArgoCD apps across 4 sync waves. Each wave gates on the previous being healthy.
```

**Post 2** (445 chars)

```
Here's what each wave does in the Crossplane GitOps setup:

• Wave 7: Crossplane core via Helm — control plane + base CRDs, single-node k3s tuning
• Wave 8: Provider CRs (Helm + Kubernetes providers from crossplane-contrib) — registers provider CRDs
• Wave 9: ProviderConfigs with InjectedIdentity — CRDs now exist, no secrets needed in local lab
• Wave 10: XRD + Composition — XCluster abstraction with patch wiring
```

**Post 3** (378 chars)

```
Running local k3s means no cloud credentials — so the Composition creates a ConfigMap instead of real EKS/GKE resources. Same abstraction, same patch mechanism, honest about the constraint.

Full post: Kubernetes GitOps Lab Part 5 — Adding Multi-Cluster Provisioning with Crossplane

[link]

#Kubernetes #Crossplane #GitOps #ArgoCD
```

---

## 📘 Facebook

```
Sharing my latest Kubernetes GitOps Lab post — this one tripped me up more than I expected.

I was trying to deploy Crossplane via ArgoCD and kept hitting sync failures. Turns out there's a CRD ordering problem: Crossplane's providers install their own CRDs, and if ArgoCD tries to apply ProviderConfigs before those CRDs exist, it fails.

The solution is splitting the deployment across 4 sync waves — something I hadn't needed to do for any of the earlier tools in this series.

The post walks through the full wave setup, InjectedIdentity ProviderConfigs for local clusters, and building an XRD + Composition even without real cloud credentials.

Has anyone else hit ordering issues with Crossplane in GitOps setups? Curious what approaches others have used.
```

---

_Hashtags chosen: `#Crossplane` and `#PlatformEngineering` added to standard set — Crossplane is the specific tool, platform engineering is the audience most likely searching for multi-cluster provisioning content._
