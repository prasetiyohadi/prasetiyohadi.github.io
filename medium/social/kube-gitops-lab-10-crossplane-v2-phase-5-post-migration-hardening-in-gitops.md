# Social Posts — Kube GitOps Lab Part 10: Crossplane v2 Phase 5

Source: `medium/kube-gitops-lab-10-crossplane-v2-phase-5-post-migration-hardening-in-gitops.md`

---

## 𝕏 Twitter/X

**Post 1 — Hook** (203 chars)

```
Thought Crossplane v2 migration was done after Part 9. Then I tried creating a Cluster claim.

Two config gaps and a wildcard MRAP later — it works. Key fix: ClusterProviderConfig ≠ ProviderConfig in v2.
```

**Post 2 — CTA** (217 chars)

```
Part 10 of Kubernetes GitOps Lab: ClusterProviderConfig, namespace patches in Compositions, MRAP for targeted resource activation.

Full walkthrough: [link] #Crossplane #GitOps #DevOps #HandsOnEngineer #KubeGitOps
```

---

## 💼 LinkedIn

```
I said Part 9 was the end of the Crossplane v2 migration. Then I tried creating an actual Cluster claim.

Three things broke or were silently wrong:
- The v2 .m.crossplane.io providers need a ClusterProviderConfig, not a ProviderConfig — Crossplane resolves the wrong kind if you don't specify it explicitly
- Without a namespace patch in the Composition, composed Objects land in crossplane-system instead of the claim's namespace
- Crossplane v2 auto-creates a wildcard MRAP that watches every CRD — fine in a lab, wasteful at scale

Part 10 of my Kubernetes GitOps Lab series closes all three. I added ClusterProviderConfig resources for both provider-helm and provider-kubernetes, updated the Composition with an explicit kind reference and a namespace patch, and replaced the wildcard MRAP with a targeted one that only activates the four resource types the lab actually uses. Then I created real Cluster claims to smoke-test the full path end-to-end.

The pattern from this entire migration series: v2 isn't one upgrade. It's a sequence of small gaps. Each phase closes one.

Full post with all the YAML: [link]

#Crossplane #Kubernetes #GitOps #ArgoCD #PlatformEngineering #DevOps #InfrastructureAsCode
```

---

## ✉️ Email Subject Lines

1. `Why ClusterProviderConfig isn't the same as ProviderConfig in Crossplane v2`
2. `I thought the migration was done — three more gaps in Crossplane v2`
3. `Crossplane v2 hardening: ClusterProviderConfig, MRAP, and namespace patches`
4. `Kubernetes GitOps Lab #10: Crossplane v2 Phase 5 — Post-Migration Hardening`

---

## ☁️ Bluesky

**Post 1 — Hook** (248 chars)

```
Thought Crossplane v2 migration was done after Part 9. First Cluster claim revealed 3 gaps: wrong provider config kind, Objects landing in wrong namespace, wildcard MRAP watching every CRD. Phase 5 closes all three.
```

**Post 2 — CTA** (238 chars)

```
Kubernetes GitOps Lab Part 10: ClusterProviderConfig for v2 APIs, namespace patch in Compositions, targeted MRAP, first real Cluster claims as smoke test. Full walkthrough: [link] #Crossplane #GitOps #KubeGitOps
```

---

## 🧵 Threads

**Post 1 — What was built** (398 chars)

```
Thought the Crossplane v2 migration was done after Part 9. Then I tried creating an actual Cluster claim.

Two config types that look identical but aren't: ProviderConfig (v1 legacy) and ClusterProviderConfig (v2). If you don't specify kind in providerConfigRef, Crossplane picks the wrong one — silently.

That was gap one. There were two more.
```

**Post 2 — What's covered** (441 chars)

```
Part 10 of Kubernetes GitOps Lab closes the remaining v2 hardening gaps:

- ClusterProviderConfig for provider-helm and provider-kubernetes (v2 cluster-scoped, InjectedIdentity)
- Composition update: explicit kind: ClusterProviderConfig + namespace patch so composed Objects land in the claim's namespace, not crossplane-system
- MRAP: replaced wildcard resource activation with a targeted policy watching only 4 resource types
```

**Post 3 — CTA** (330 chars)

```
Then I created two real Cluster claims to verify the full path end-to-end.

The migration series takeaway: v2 isn't one upgrade. It's a sequence of small gaps — version hops, provider APIs, scoping, activation policies. Each phase is small. Each builds on the last.

Full post: [link]
```

---

## 📘 Facebook

```
Just posted Part 10 of my Kubernetes GitOps Lab series — and it starts with a confession: I thought the Crossplane v2 migration was done.

Then I actually tried creating a Cluster claim. Turns out there were still three gaps to close: a provider config scoping issue specific to v2 APIs, composed Objects landing in the wrong namespace, and Crossplane watching every CRD in the cluster by default.

The post walks through ClusterProviderConfig, a namespace patch in the Composition, a targeted ManagedResourceActivationPolicy, and first real Cluster claims as a smoke test.

Full post: [link]

Have you run into the ClusterProviderConfig vs ProviderConfig confusion in Crossplane v2? What tripped you up most during the migration?
```

---

_Hashtag notes: `#Crossplane` and `#GitOps` are the primary topic tags. `#KubeGitOps` used on X and Bluesky as a series thread tag. `#HandsOnEngineer` on X as the personal brand tag. LinkedIn uses `#PlatformEngineering` and `#InfrastructureAsCode` to reach the broader IaC/platform audience beyond GitOps-specific followers._
