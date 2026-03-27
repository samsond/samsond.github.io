---
layout: post
title: "Why Tags Fail as Trust Anchors"
date: 2026-03-25 21:00:00 -0700
categories: security
description: "Tags are mutable references. Full commit SHAs are the safer trust anchor for GitHub Actions."
tags: [github-actions, security, supply-chain, ci-cd]
comments: false
---

## Executive Summary

A GitHub Action reference such as `uses: some-org/action@v1.2.3` looks fixed, but it is not immutable. A tag is a name that resolves to a commit when the workflow runs. If we want deterministic and auditable execution, third-party actions should be pinned to full commit SHAs rather than tags.

## The Core Distinction

GitHub's own guidance is explicit on this point: a full commit SHA is the immutable way to reference an action. Tags and branches are different. They are moving references that require trust in whoever controls them.

That distinction matters because a workflow can begin executing different code even though nothing in the workflow file has changed. The configuration still looks reviewed and stable, while the effective dependency is no longer the one we think it is. GitHub makes the same distinction for reusable workflows: a commit SHA is fixed, while a tag or branch is not.

## Why This Matters Operationally

The risk is not theoretical. In the `tj-actions` compromise, attackers pushed new Git tags so existing references resolved to a malicious commit and downstream workflows pulled altered code. That is the practical failure mode: execution changes without a visible dependency update on the consumer side.

The broader issue is structural. We often treat tags as if they were immutable release artifacts, but they behave as mutable references. OpenSSF's workflow-hardening guidance recommends hash-pinning actions precisely because tag-based references do not provide immutability.

## The Industry Already Has the Right Mental Model

This pattern is not unique to GitHub Actions. Docker documents the same distinction in container distribution: tags can be reused or changed, while digests are immutable and ensure the exact same image is pulled every time. The industry already has a working mental model here. Names are convenient, but hashes carry identity.

## Pinning Is Necessary, Not Sufficient

In CI, the consequences are sharper because workflows often run with access to secrets, tokens, and deployment paths. If a third-party action is compromised or repointed, the blast radius is shaped not only by what code runs, but by what privileges that code receives.

For that reason, pinning to a full commit SHA should be paired with limiting the permissions, secrets, and access granted to each action to only what it requires.

## Recommendation

If we want deterministic and auditable execution, we should pin third-party actions to full commit SHAs rather than tags, and treat tag-based references as discovery labels rather than trust anchors.

## References

1. [GitHub Docs, Secure use reference](https://docs.github.com/en/actions/reference/security/secure-use)
2. [OpenSSF, Mitigating Attack Vectors in GitHub Workflows](https://openssf.org/blog/2024/08/12/mitigating-attack-vectors-in-github-workflows/)
3. [Palo Alto Networks Unit 42, GitHub Actions Supply Chain Attack](https://unit42.paloaltonetworks.com/github-actions-supply-chain-attack/)
4. [CrowdStrike, From Scanner to Stealer: Inside the trivy-action Supply Chain Compromise](https://www.crowdstrike.com/en-us/blog/from-scanner-to-stealer-inside-the-trivy-action-supply-chain-compromise/)
5. [Docker Docs, Image digests](https://docs.docker.com/dhi/core-concepts/digests/)
