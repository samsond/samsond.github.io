---
layout: post
title:  "Making Sense of Kubernetes Rolling Updates"
date:   2025-12-19 10:00:00 -0800
categories: kubernetes
description: "Master maxUnavailable and maxSurge—the math behind zero-downtime Kubernetes deployments. Understand constraints, rounding surprises, and the capacity vs. risk tradeoff."
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

**Definition:** The maximum number of Pods that can be *unavailable* (terminated or not yet ready) at any point during the rollout.

If `spec.replicas = 10` and `maxUnavailable = 2`, then **at least 8 Pods must be available** (10 - 2 = 8). If you use a percentage like `20%`, Kubernetes calculates it relative to replicas.

`maxUnavailable` is about **risk**: higher values mean faster rollout but more capacity loss; lower values mean slower rollout but safer margins.

---

## Constraint 2: maxSurge

**Definition:** The maximum number of *extra* Pods (beyond `spec.replicas`) that can exist simultaneously during the rollout.

If `spec.replicas = 10` and `maxSurge = 3`, then up to **13 Pods can run at the same time** (10 + 3), giving Kubernetes time to verify new Pods are healthy before removing old ones.

`maxSurge` is about **cost**: higher values mean faster rollout but more CPU/memory consumed; lower values mean tighter resource use but slower rollout.

---

## Percent Becomes Pods

This is where asymmetric rounding creates surprises.

Kubernetes uses **different rounding for each field**:

- **`maxUnavailable`** (percentage) → **floor** (round down)
- **`maxSurge`** (percentage) → **ceiling** (round up)

When you write `maxUnavailable: 25%` with 10 replicas:

```
Unavailable Pods = floor(10 × 0.25) = floor(2.5) = 2
```

But when you write `maxSurge: 25%` with 10 replicas:

```
Surge Pods = ceiling(10 × 0.25) = ceiling(2.5) = 3
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

### Example 2: 3 Replicas (The Asymmetric Rounding Case)

```
replicas:        3
maxUnavailable:  25%  →  floor(3 × 0.25) = floor(0.75) = 0 Pods
maxSurge:        30%  →  ceiling(3 × 0.30) = ceiling(0.9) = 1 Pod

During rollout:
- Minimum available: 3 - 0 = 3 Pods (no unavailability allowed!)
- Maximum running:   3 + 1 = 4 Pods
```

**Important:** maxUnavailable floors to 0. With only 3 replicas, 25% floors to 0.75 → 0. This means **all 3 Pods must stay available** during the rollout. The constraint is effectively "keep all pods running."

However, Kubernetes has a validity rule: **if both maxUnavailable and maxSurge resolve to 0, maxUnavailable is forced to 1**. So in practice, at least 1 Pod can become unavailable, otherwise rollouts couldn't progress.

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

### Example 4: 4 Replicas (The Flooring Case)

```
replicas:        4
maxUnavailable:  20%  →  floor(4 × 0.20) = floor(0.8) = 0 Pods
maxSurge:        20%  →  ceiling(4 × 0.20) = ceiling(0.8) = 1 Pod

During rollout (with validity fallback):
- Minimum available: 4 - 1 = 3 Pods (clamped because 0 would violate validity)
- Maximum running:   4 + 1 = 5 Pods
```

**Subtle:** You asked for 20%, but maxUnavailable floors to 0.8 → 0. Kubernetes then applies the validity rule: if maxUnavailable is 0 and maxSurge is non-zero, allow the rollout. But if both would be 0, force maxUnavailable to 1.

---

## Why Asymmetric Rounding Matters

**maxUnavailable floors (rounds down):** Kubernetes prefers to keep more pods available when unsure. If 25% of 3 is 0.75, it floors to 0, meaning "try not to take any down."

**maxSurge ceilings (rounds up):** Kubernetes prefers to allow more surge when unsure. If 25% of 3 is 0.75, it ceilings to 1, meaning "allow one extra pod to speed up verification."

When you have **small replica counts** (3–10 Pods):
- Floor and ceiling diverge visibly
- maxUnavailable may become 0 (triggering the validity rule)
- **Use absolute counts for precise control**

When you have **large replica counts** (50+ Pods):
- Rounding error is negligible (0.4 pod out of 50 is invisible)
- Percentages work well and scale automatically
- Either approach is fine

---

## Real-World Examples: Choosing Constraints

Both fields default to 25%. On a 10-pod deployment, that's 2 pods unavailable (floor), 2 pods surge (ceiling). On a 4-pod deployment, that's 0 pods unavailable (floored), 1 pod surge (ceiled), then bumped to 1 unavailable by the validity rule.

Suppose you run a service with **15 Pods** handling user traffic:

### Conservative Approach (Safety First)
```yaml
maxUnavailable: 0        # Always keep all 15 serving
maxSurge: 5              # Temporary max 20 pods
```
- **Rollout time:** ~15 minutes (5 pods in parallel, all old pods stay up)
- **Capacity risk:** Zero (no pods ever unavailable)
- **Cost spike:** Medium (5 extra pods temporarily)
- **Use case:** Critical payment service, strict SLA

### Balanced Approach
```yaml
maxUnavailable: 3        # Always keep ≥12 serving (20% headroom)
maxSurge: 5              # Temporary max 20 pods
```
- **Rollout time:** ~10 minutes (3–5 pods per wave)
- **Capacity risk:** Moderate
- **Cost spike:** Medium
- **Use case:** Standard web service, normal SLA

### Aggressive Approach (Speed First)
```yaml
maxUnavailable: 25%      # floor(15 × 0.25) = 3 pods → keep ≥12 available
maxSurge: 50%            # ceiling(15 × 0.50) = 8 pods → temporary max 23 pods
```
- **Rollout time:** ~3 minutes (rapid waves)
- **Capacity risk:** Higher (down to 12 pods, ~20% loss)
- **Cost spike:** Significant (8 extra pods temporarily)
- **Use case:** Batch processing, non-critical service, good availability buffer

---

## Connecting the Pieces: What Else You Need

This post covered the math. But to make your rollout strategy work reliably in production, you need:

- **Pod Disruption Budgets (PDB):** Protect against voluntary evictions during cluster maintenance
- **Readiness & Startup Probes:** Ensure new pods are truly ready before counting them as available
- **minReadySeconds:** Stabilize flaky pods before trusting them
- **Cluster Headroom & Autoscaling:** Guarantee resources exist for your surge capacity
- **Topology Spread & Anti-Affinity:** Distribute pods across nodes/zones to avoid correlated failures

Each of these deserves its own post. For now, know this: the `maxUnavailable` and `maxSurge` fields define the *guardrails*, but the supporting infrastructure (PDB, probes, affinity, autoscaling) makes the guarantee real.

---

## Key Takeaways

1. **`maxUnavailable` controls risk** (and floors): How much capacity can you afford to lose during rollout?
2. **`maxSurge` controls cost** (and ceilings): How much resource overhead can you absorb temporarily?
3. **Asymmetric rounding:** maxUnavailable floors (safer), maxSurge ceilings (faster).
4. **Small replica counts expose rounding:** 25% of 4 pods is 1 with ceiling, 0 with floor. Use absolute counts for precision.
5. **Validity constraint:** maxUnavailable and maxSurge cannot both be 0. If both resolve to 0, maxUnavailable is forced to 1.
6. **These are constraints, not timers:** They shape the *pace* and *capacity envelope* of the rollout, not its absolute duration.

Your rollout strategy isn't magic—it's a deliberate choice about which risk you accept: capacity loss or cost increase. Once you see the math, you can make that choice with confidence.

---

## Resources

- [Kubernetes Rolling Update Strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy)
- [Managing Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
