---
layout: post
title:  "Making Sense of Kubernetes Rolling Updates"
date:   2025-12-19 10:00:00 -0800
categories: kubernetes
description: "Master maxUnavailable and maxSurge—the math behind zero-downtime Kubernetes deployments."
tags: [kubernetes, devops, system-architecture]
comments: false
last_modified_at: 2025-12-19 10:00:00 -0800
---

## Executive Summary

Your rollout strategy is a pair of guardrails: how many Pods may disappear, and how many extra you're allowed to temporarily pay for.

Those guardrails are `maxUnavailable` and `maxSurge`—two fields in your Kubernetes Deployment spec that control the pace of rolling updates. Yet they're often left at defaults, adjusted without understanding, or misinterpreted as "percentages only" when they actually work with both percentages *and* absolute pod counts.

In this post, we'll:

- Define both constraints relative to `spec.replicas`
- Show how percentages become pod counts
- Walk through 4 replica examples, including rounding surprises
- Connect the math to real-world capacity and risk decisions

By the end, you'll stop treating these fields as magic numbers and start using them as intentional deployment contracts.

---

## Constraint 1: maxUnavailable

**Definition:** The maximum number of Pods allowed to be unavailable at any point during a rolling update. A Pod is "unavailable" if it is not counted as Available in availableReplicas—that is, not Ready, or Ready but not yet past minReadySeconds (if set). Pods being terminated are unavailable once they stop being Ready/Available.

If `spec.replicas = 10` and `maxUnavailable = 2`, then at least 8 Pods must be available (10 - 2 = 8). If you use a percentage like `25%`, Kubernetes calculates it with floor rounding.

Note: Available means Ready, and if minReadySeconds is set, Ready for at least that many seconds before counting toward availableReplicas.

`maxUnavailable` is about risk: higher values mean faster rollout but temporary capacity loss; lower values mean slower rollout but stricter availability guarantees.

---

## Constraint 2: maxSurge

**Definition:** The maximum number of *extra* Pods (beyond `spec.replicas`) that can exist simultaneously during the rollout.

If `spec.replicas = 10` and `maxSurge = 3`, then up to 13 Pods can run at the same time (10 + 3), giving Kubernetes time to verify new Pods are healthy before removing old ones.

`maxSurge` is about cost: higher values mean faster rollout but more CPU/memory consumed; lower values mean tighter resource use but slower rollout.

---

## Percent Becomes Pods

This is where asymmetric rounding creates surprises.

Kubernetes uses different rounding for each field:

- **`maxUnavailable`** (percentage) → **round down** (via `math.Floor`)
- **`maxSurge`** (percentage) → **round up** (via `math.Ceil`)

When you write `maxUnavailable: 25%` with 10 replicas:

```
Unavailable Pods = floor(10 × 0.25) = 2
```

But when you write `maxSurge: 25%` with 10 replicas:

```
Surge Pods = ceiling(10 × 0.25) = 3
```

This asymmetry is intentional: maxSurge errs on the side of faster rollout (more surge allowed), while maxUnavailable errs on the side of safety (fewer pods unavailable). Let's see why this matters:

### Example 1: 10 Replicas (The Easy Case)

```
replicas:        10
maxUnavailable:  20%  →  floor(10 × 0.20) = 2 Pods
maxSurge:        30%  →  ceiling(10 × 0.30) = 3 Pods

During rollout:
- Minimum available: 10 - 2 = 8 Pods
- Maximum running:   10 + 3 = 13 Pods
```

With larger replica counts, the rounding is invisible. Both fields behave predictably.

### Example 2: 3 Replicas (The Flooring Case)

```
replicas:        3
maxUnavailable:  25%  →  floor(3 × 0.25) = floor(0.75) = 0 Pods
maxSurge:        30%  →  ceiling(3 × 0.30) = ceiling(0.9) = 1 Pod

During rollout:
- Minimum available: 3 - 0 = 3 Pods (all must stay available)
- Maximum running:   3 + 1 = 4 Pods
```

Important: maxUnavailable floors to 0. With only 3 replicas, 25% of 3 is 0.75, which floors to 0. This means all 3 Pods must remain available at all times. However, Kubernetes has a fencepost safeguard: if both maxUnavailable and maxSurge resolve to 0 (meaning no pods could transition and the rollout would stall), maxUnavailable is forced to 1. In this example, since maxSurge > 0, the rollout can proceed even with maxUnavailable = 0—new pods start first, and old pods are only terminated after new ones are available.

### Example 3: 5 Replicas (The Different-Rounding Case)

```
replicas:        5
maxUnavailable:  30%  →  floor(5 × 0.30) = floor(1.5) = 1 Pod
maxSurge:        25%  →  ceiling(5 × 0.25) = ceiling(1.25) = 2 Pods

During rollout:
- Minimum available: 5 - 1 = 4 Pods
- Maximum running:   5 + 2 = 7 Pods
```

Notice: maxUnavailable floors to 1, but maxSurge ceilings to 2. The asymmetry is visible: you're allowing more surge (2 extra pods) than unavailability (1 pod down). This is Kubernetes being conservative—it prefers brief overprovision to risking availability.

### Example 4: 4 Replicas (The Rounding Surprise)

```
replicas:        4
maxUnavailable:  25%  →  floor(4 × 0.25) = floor(1) = 1 Pod
maxSurge:        25%  →  ceiling(4 × 0.25) = ceiling(1) = 1 Pod

During rollout:
- Minimum available: 4 - 1 = 3 Pods
- Maximum running:   4 + 1 = 5 Pods
```

With 4 replicas and 25%, the calculation is exact: 25% of 4 = 1 pod, so both floor and ceiling land on 1. This shows symmetric rounding when the percentage aligns with the replica count.

---

## Why Asymmetric Rounding Matters

maxUnavailable rounds down (via `math.Floor`): Kubernetes prefers to keep more pods available when unsure. If 25% of 3 is 0.75, it rounds down to 0, meaning "minimize unavailability."

maxSurge rounds up (via `math.Ceil`): Kubernetes prefers to allow more surge when unsure. If 25% of 3 is 0.75, it rounds up to 1, meaning "allow extra capacity to verify new pods."

When you have small replica counts (3–10 Pods):
- Floor and ceiling can diverge significantly
- maxUnavailable may resolve to 0, requiring all pods to stay available
- Use absolute counts for precise control

When you have large replica counts (50+ Pods):
- Rounding error becomes negligible (1 pod out of 100 is 1%)
- Percentages work well and scale naturally
- Either approach is reasonable

---

## Real-World Examples: Choosing Constraints

Both fields default to 25%. On a 10-pod deployment, that's 2 pods unavailable (floor), 3 pods surge (ceiling). This asymmetry means you get stricter availability (2 down) but more generous surge room (3 extra).

Suppose you run a service with 15 Pods handling user traffic:

### Conservative Approach (Safety First)
```yaml
maxUnavailable: 0        # Always keep all 15 available
maxSurge: 5              # Temporary max 20 pods
```
- **Rollout pace:** Slower (5 new pods start, old pods terminated only after new ones become available)
- **Capacity risk:** No reduction in available replicas from the rollout strategy; if new pods fail readiness, the rollout stalls rather than scaling down old pods
- **Cost spike:** Medium (5 extra pods temporarily)
- **Use case:** Critical payment service, strict SLA

### Balanced Approach
```yaml
maxUnavailable: 3        # Always keep ≥12 available
maxSurge: 5              # Temporary max 20 pods
```
- **Rollout pace:** Moderate (3–5 pods transition per wave)
- **Capacity risk:** Moderate (down to 12 pods, ~20% less capacity)
- **Cost spike:** Medium
- **Use case:** Standard web service, normal SLA

### Aggressive Approach (Speed First)
```yaml
maxUnavailable: 25%      # floor(15 × 0.25) = 3 pods → keep ≥12 available
maxSurge: 50%            # ceiling(15 × 0.50) = 8 pods → temporary max 23 pods
```
- **Rollout pace:** Faster (larger waves transition in parallel)
- **Capacity risk:** Higher (down to 12 pods, ~20% less capacity)
- **Cost spike:** Significant (8 extra pods temporarily)
- **Use case:** Batch processing, non-critical service, good availability buffer

---

## Connecting the Pieces: What Else You Need

This post covered the math. But to make your rollout strategy work reliably in production, you need:

- **Pod Disruption Budgets (PDB):** Protect pods from voluntary evictions during cluster maintenance (node drains, upgrades). Note: PDBs do not constrain Deployment-driven rolling updates—those are governed by maxUnavailable and maxSurge.
- **Readiness Probes:** Readiness probes determine when a pod transitions to Ready state. Once Ready, a pod is subject to minReadySeconds (if set) before being counted as Available in the rolling update math.
- **minReadySeconds:** Gates when a Ready pod becomes Available for rollout calculations. A pod must remain Ready for N seconds before counting toward availableReplicas (if minReadySeconds is set). This prevents "flaky ready" pods from being trusted immediately.
- **Cluster Headroom & Autoscaling:** Guarantee resources exist for your surge capacity. If your cluster runs out of CPU/memory, surge pods will remain pending and your rollout will stall.
- **Topology Spread & Anti-Affinity:** Distribute pods across nodes/zones to avoid correlated failures. If all surge pods land on the same node, a single node failure cascades into multiple pod losses.

Each of these deserves its own post. For now, know this: the `maxUnavailable` and `maxSurge` fields define the guardrails, but the supporting infrastructure (readiness probes, minReadySeconds, PDB, cluster resources, affinity) makes the guarantee real.

---

## Key Takeaways

1. **`maxUnavailable` controls risk** (rounds down): How much capacity loss can you afford during rollout?
2. **`maxSurge` controls cost** (rounds up): How much temporary resource overage can you absorb?
3. **Asymmetric rounding:** maxUnavailable rounds down (conservative), maxSurge rounds up (permissive).
4. **Small replica counts expose rounding:** With 3–10 pods, floor and ceiling diverge visibly. Use absolute counts for precision.
5. **Fencepost safeguard:** maxUnavailable and maxSurge cannot both be 0. If both would resolve to 0, maxUnavailable is forced to 1 to allow rollout to proceed.
6. **"Available" means Ready and past minReadySeconds:** A pod is available only after passing readiness probes and remaining Ready for minReadySeconds (if set). Plan your probes and minReadySeconds accordingly.
7. **These are constraints, not timers:** They shape the *pace* and *capacity envelope* of the rollout, not its absolute duration. Actual rollout time depends on how many pods can transition per wave, how long each pod takes to start and become ready, and your cluster's resource availability.

Your rollout strategy isn't magic—it's a deliberate choice about which risk you accept: capacity loss, cost increase, or rollout speed. Once you see the math and validate it with production infrastructure, you can execute deployments with confidence.

---

## Resources

- [Kubernetes Rolling Update Strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy)
- [Managing Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [IntOrString Rounding: Round Up / Round Down Logic](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/apimachinery/pkg/util/intstr/intstr.go#L182)
- [Kubernetes Deployment Controller: Fencepost Safeguard](https://github.com/kubernetes/kubectl/blob/master/pkg/util/deployment/deployment.go#L248)
