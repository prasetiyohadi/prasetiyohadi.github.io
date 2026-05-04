# Social Posts: Kube GitOps Lab #9 — Crossplane v2 Phase 4: Upgrading Providers and Adopting Namespace-Scoped Objects

---

## 𝕏 Twitter/X

**Post 1** (hook)

```
Crossplane core was on v2.2. Providers were still crossplane-contrib v0.x.

Phase 4: upgrade to upbound/provider-helm:v1.2.1 and upbound/provider-kubernetes:v1.2.2.

New providers didn't go healthy. Reason: old ProviderRevision still present, conflicting on init. ArgoCD can't prune what Crossplane created.

Manual kubectl delete. Done in 60s.
```

_(268 chars)_

**Post 2** (CTA)

```
Also hit XRD spec.scope being immutable — tried Namespaced, got "Value is immutable" from ArgoCD.

Full post covers provider org migration, ProviderRevision cleanup, LegacyCluster scope, and the m-prefixed Object API.

Kube GitOps Lab #9: [link]

#Kubernetes #Crossplane #GitOps #DevOps #HandsOnEngineer
```

_(280 chars)_

---

## 💼 LinkedIn

```
After four posts migrating Crossplane core to v2.2, the providers were still on the old crossplane-contrib packages — v0.19 for helm, v0.15 for kubernetes. Upbound now publishes these officially under their own registry namespace at v1.x, with new API groups. Phase 4 closes that gap.

In my latest Kube GitOps Lab post, I walk through:

• Upgrading provider-helm and provider-kubernetes from crossplane-contrib to the Upbound-published v1.x releases — and why the new providers got stuck in a non-healthy state (ProviderRevision conflicts, manual cleanup required)
• Setting spec.scope: LegacyCluster on the XRD — a field I didn't know was immutable until ArgoCD threw a validation error trying to set it to Namespaced
• Switching the Composition Object API from kubernetes.crossplane.io/v1alpha2 to kubernetes.m.crossplane.io/v1alpha1 — the namespace-scoped variant the updated provider ships
• The decision to keep v1 XRD (claimNames intact) rather than migrating to apiextensions.crossplane.io/v2

The broader pattern across all four migration phases: GitOps handles desired state, but not cleanup of objects Crossplane creates outside of Git. ProviderRevisions, orphaned CRD fields — those need targeted manual steps. Knowing which operations fall inside the reconciliation loop vs. outside it is the practical skill.

Kube GitOps Lab, Part 9 — link in comments.

#Kubernetes #GitOps #Crossplane #ArgoCD #DevOps #PlatformEngineering #InfrastructureAsCode
```

_(1,347 chars)_

---

## ✉️ Email Subject Lines

1. **Curiosity**: Why did my Crossplane providers get stuck after upgrading to v2?
2. **How I...**: How I migrated from crossplane-contrib to upbound providers (and the two things that broke)
3. **Direct**: Crossplane v2 provider upgrades: ProviderRevision conflicts and immutable XRD scope
4. **Series**: Kube GitOps Lab #9: Crossplane v2 Phase 4 — Provider Upgrades

---

## ☁️ Bluesky

**Post 1**

```
Crossplane core: v2.2. Providers: still crossplane-contrib v0.x.

Phase 4 fixes that — upgrade to upbound/provider-helm:v1.2.1 and provider-kubernetes:v1.2.2.

Gotcha: old ProviderRevision conflicts on init. ArgoCD can't prune it. Manual delete required.
```

_(274 chars)_

**Post 2**

```
Also: XRD spec.scope is immutable. Tried Namespaced, got a validation error. Set LegacyCluster instead.

And switched the Composition Object API to kubernetes.m.crossplane.io/v1alpha1 (namespace-scoped).

Kube GitOps Lab #9: [link]
```

_(233 chars)_

---

## 🧵 Threads

**Post 1**

```
I spent four posts migrating Crossplane core to v2.2. The providers were still on crossplane-contrib v0.x the whole time.

Phase 4 is about closing that gap: upgrade to the Upbound-published v1.x releases and switch the Composition to use the new namespace-scoped Object API.

Two things broke immediately. Worth documenting.
```

_(330 chars)_

**Post 2**

```
What this post covers:

• Provider org migration: crossplane-contrib → upbound, v0.19 → v1.2.1 (helm), v0.15 → v1.2.2 (kubernetes)
• ProviderRevision conflicts — new providers stuck non-healthy until old revision manually deleted (ArgoCD can't prune what Crossplane creates)
• XRD spec.scope immutability — tried Namespaced, it's immutable, LegacyCluster is the right value
• Composition Object API: switched to kubernetes.m.crossplane.io/v1alpha1 for namespace-scoped Objects
```

_(489 chars)_

**Post 3**

```
The pattern across all four migration phases: GitOps handles desired state, not garbage collection of objects Crossplane manages outside Git.

Knowing that boundary is the practical skill — not the YAML.

Kube GitOps Lab, Part 9: [link]

#Kubernetes #Crossplane #GitOps #ArgoCD
```

_(282 chars)_

---

## 📘 Facebook

```
Just posted Part 9 of my Kubernetes GitOps Lab series — this one covers the last piece of the Crossplane v2 migration I hadn't touched yet: upgrading the providers themselves.

After getting Crossplane core to v2.2, provider-helm and provider-kubernetes were still on old crossplane-contrib packages. Switching to the Upbound-published v1.x versions sounds straightforward, but there's a ProviderRevision conflict that blocks the new providers from going healthy — and ArgoCD can't fix it automatically because the stale revision lives outside of Git.

Also ran into an immutable XRD field. Fun times.

If you're working through a Crossplane v2 migration, the two gotchas in this post might save you some debugging time. Have you hit ProviderRevision issues before?

[link]
```

---

_Hashtags chosen: `#Kubernetes #Crossplane #GitOps #ArgoCD #DevOps #PlatformEngineering #InfrastructureAsCode` — all directly referenced tools or categories in the post. `#HandsOnEngineer` used on Twitter to match series brand._
