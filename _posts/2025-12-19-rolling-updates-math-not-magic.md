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

`maxUnavailable` and `maxSurge` are two fields in your Kubernetes Deployment spec that control the pace of rolling updates. They define your rollout strategy: how many Pods can disappear at once, and how many extra Pods you're willing to pay for temporarily.

These fields are often left at defaults or adjusted without understanding. Most teams treat them as "percentages only," missing that they work with both percentages *and* absolute pod counts. Worse, different rounding rules apply to each field—asymmetry that surprises teams with small replica counts.

In this post:

- Define both constraints relative to `spec.replicas`
- Show how percentages become pod counts and why rounding matters
- Walk through 4 replica examples with rounding outcomes
- Connect the math to real-world capacity and risk decisions

After reading, you'll understand the math behind your rollout strategy and make intentional deployment decisions.

---

## Constraint 1: maxUnavailable

**Definition:** The maximum number of Pods allowed to be unavailable during a rolling update. A Pod is unavailable if not counted as Available in `availableReplicas`—meaning not yet Ready, or Ready but not yet past `minReadySeconds`.

If `spec.replicas = 10` and `maxUnavailable = 2`, at least 8 Pods must remain available. With a percentage like `25%`, Kubernetes calculates using floor rounding (toward fewer unavailable Pods).

**Why it matters:** `maxUnavailable` controls availability risk. Higher values speed rollout but reduce available capacity during deployment; lower values maintain availability but slow rollout.

---

## Constraint 2: maxSurge

**Definition:** The maximum number of *extra* Pods (beyond `spec.replicas`) allowed simultaneously during the rollout.

If `spec.replicas = 10` and `maxSurge = 3`, up to 13 Pods run concurrently. New Pods start before old ones terminate, allowing Kubernetes to verify health before removal.

**Why it matters:** `maxSurge` controls cost and speed. Higher values enable faster rollout and easier health verification but require more CPU/memory; lower values reduce resource spikes but slow deployment.

---

## Percent Becomes Pods: Asymmetric Rounding

Kubernetes applies different rounding rules to each field:

- **`maxUnavailable`** (percentage) → **floor** (round down)
- **`maxSurge`** (percentage) → **ceiling** (round up)

With `maxUnavailable: 25%` and 10 replicas:
```
Unavailable Pods = floor(10 × 0.25) = 2
```

With `maxSurge: 25%` and 10 replicas:
```
Surge Pods = ceiling(10 × 0.25) = 3
```

This asymmetry is intentional. `maxUnavailable` floors to prioritize safety; `maxSurge` ceilings to prioritize faster verification. The impact becomes visible with small replica counts:

### Example 1: 10 Replicas

```
replicas:        10
maxUnavailable:  20%  →  floor(10 × 0.20) = 2 Pods
maxSurge:        30%  →  ceiling(10 × 0.30) = 3 Pods

During rollout:
- Minimum available: 10 - 2 = 8 Pods
- Maximum running:   10 + 3 = 13 Pods
```

With larger replica counts, rounding is negligible. Both fields behave predictably.

### Example 2: 3 Replicas (Flooring Impact)

```
replicas:        3
maxUnavailable:  25%  →  floor(3 × 0.25) = floor(0.75) = 0 Pods
maxSurge:        30%  →  ceiling(3 × 0.30) = ceiling(0.9) = 1 Pod

During rollout:
- Minimum available: 3 - 0 = 3 Pods (all must stay available)
- Maximum running:   3 + 1 = 4 Pods
```

`maxUnavailable` floors to 0 when the percentage calculates below 1. Here, 25% of 3 equals 0.75, which floors to 0. All 3 Pods must remain available simultaneously. Kubernetes enforces a safeguard: if both fields resolve to 0, the rollout would stall (no pods could transition). In that case, `maxUnavailable` is forced to 1. Here, since `maxSurge = 1 > 0`, new Pods can start before old ones terminate, allowing the rollout to proceed.

### Example 3: 5 Replicas (Asymmetry Visible)

```
replicas:        5
maxUnavailable:  30%  →  floor(5 × 0.30) = floor(1.5) = 1 Pod
maxSurge:        25%  →  ceiling(5 × 0.25) = ceiling(1.25) = 2 Pods

During rollout:
- Minimum available: 5 - 1 = 4 Pods
- Maximum running:   5 + 2 = 7 Pods
```

`maxUnavailable` floors to 1, but `maxSurge` ceilings to 2. The asymmetry is now visible: surge exceeds unavailability (2 vs. 1). Kubernetes prioritizes brief overprovision over risking availability.

### Example 4: 4 Replicas (Symmetric Rounding)

```
replicas:        4
maxUnavailable:  25%  →  floor(4 × 0.25) = 1 Pod
maxSurge:        25%  →  ceiling(4 × 0.25) = 1 Pod

During rollout:
- Minimum available: 4 - 1 = 3 Pods
- Maximum running:   4 + 1 = 5 Pods
```

When the percentage aligns with the replica count (25% of 4 = 1), both floor and ceiling resolve identically.

---

## Why Asymmetric Rounding Matters

`maxUnavailable` floors: Kubernetes minimizes availability risk. If 25% of 3 is 0.75, it floors to 0 (preserve all available Pods).

`maxSurge` ceilings: Kubernetes maximizes rollout confidence. If 25% of 3 is 0.75, it ceilings to 1 (allow extra capacity to verify new Pods).

**Small replica counts (3–10):** Floor and ceiling diverge noticeably. `maxUnavailable` may resolve to 0, requiring all Pods to remain available. Use absolute counts (integers) for precise control.

**Large replica counts (50+):** Rounding error becomes negligible. Percentages scale naturally and both approaches work well.

---

## Real-World Examples: Choosing Constraints

Both fields default to 25%. On 10 Pods, this translates to 2 Pods unavailable and 3 Pods surge—stricter availability but more generous surge room.

For a service with 15 Pods:

### Conservative (Safety First)
```yaml
maxUnavailable: 0        # All 15 Pods always available
maxSurge: 5              # Temporary max 20 Pods
```
- **Pace:** Slower—new Pods start before old ones terminate
- **Availability risk:** None—rollout doesn't reduce available Pods
- **Cost:** Medium (5 extra Pods temporarily)
- **Use case:** Critical payment service with strict SLA

### Balanced
```yaml
maxUnavailable: 3        # Keep ≥12 available (20% capacity loss)
maxSurge: 5              # Temporary max 20 Pods
```
- **Pace:** Moderate—3–5 Pods transition per wave
- **Availability risk:** Moderate (20% reduction)
- **Cost:** Medium (5 extra Pods temporarily)
- **Use case:** Standard web service with normal SLA

### Aggressive (Speed First)
```yaml
maxUnavailable: 25%      # floor(15 × 0.25) = 3 → keep ≥12 available
maxSurge: 50%            # ceiling(15 × 0.50) = 8 → temporary max 23 Pods
```
- **Pace:** Faster—larger waves transition in parallel
- **Availability risk:** Higher (20% reduction)
- **Cost:** Significant (8 extra Pods temporarily)
- **Use case:** Batch processing or non-critical services with headroom

---

## Beyond maxUnavailable and maxSurge

The math defines the guardrails, but production reliability requires supporting infrastructure:

- **Readiness Probes:** Determine when Pods transition to Ready. A Pod becomes Available only after reaching Ready state and staying Ready for `minReadySeconds` (if set).
- **minReadySeconds:** Prevent "flaky ready" Pods from being trusted immediately. A Pod must remain Ready for N seconds before counting toward `availableReplicas`.
- **Pod Disruption Budgets (PDB):** Protect Pods from voluntary evictions during cluster maintenance (node drains, upgrades). Note: PDBs do not govern Deployment rolling updates—`maxUnavailable` and `maxSurge` do.
- **Cluster Headroom:** Ensure resources exist for surge Pods. Without sufficient CPU/memory, surge Pods remain pending and the rollout stalls.
- **Topology Spread & Anti-Affinity:** Distribute Pods across nodes/zones to prevent correlated failures. If all surge Pods land on one node, a single failure cascades into multiple losses.

Each of these warrants deeper exploration. The key: `maxUnavailable` and `maxSurge` set the constraints, but the infrastructure makes them reliable.

---

## Key Takeaways

1. **`maxUnavailable` controls availability risk** (rounds down): How many Pod failures can you afford before the service degrades?
2. **`maxSurge` controls cost and speed** (rounds up): How much temporary resource overage can you accept?
3. **Asymmetric rounding is intentional:** `maxUnavailable` floors (safety), `maxSurge` ceilings (speed).
4. **Small replica counts expose rounding impact:** With 3–10 Pods, use absolute counts for precise control.
5. **Fencepost safeguard:** If both fields resolve to 0, `maxUnavailable` is forced to 1 to prevent rollout stall.
6. **"Available" requires Ready + minReadySeconds:** A Pod becomes available only after readiness probes pass and it remains Ready for `minReadySeconds` (if set).
7. **Constraints shape pace and capacity, not duration:** Actual rollout time depends on pod transition rate, startup time, readiness duration, and cluster resource availability.

Your rollout strategy isn't magic—it's a deliberate choice about which risk you accept: capacity loss, cost increase, or rollout speed. Once you see the math and validate it with production infrastructure, you can execute deployments with confidence.

---

## Resources

- [Kubernetes Rolling Update Strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy)
- [Managing Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [IntOrString Rounding: Round Up / Round Down Logic](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/apimachinery/pkg/util/intstr/intstr.go#L182)
- [Kubernetes Deployment Controller: Fencepost Safeguard](https://github.com/kubernetes/kubectl/blob/master/pkg/util/deployment/deployment.go#L248)
