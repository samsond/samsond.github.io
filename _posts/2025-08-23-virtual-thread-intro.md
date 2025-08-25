---
layout: post
title:  "Concurrency Foundations: Virtual Threads vs CompletableFuture"
date:   2025-08-23 8:00:00 -0800
categories: concurrency
description: "Guidelines for when to use Virtual Threads or CompletableFuture in your applications"
tags: [java]
comments: false
---

## Executive summary

When you hear "concurrency in Java" your mind might first jump to `ExecutorService`, `CompletableFuture`, or maybe even the good old Thread. But with Project Loom [JEP 425](https://openjdk.org/jeps/425), Java introduced something game-changing: virtual threads.

Before we can compare virtual threads to CompletableFuture, let’s establish the basics.

## The Problem: Concurrency Has Always Been Expensive

Threads in Java are backed by OS threads. This means:

* Heavyweight: Each thread consumes memory (stack space ~1MB by default) and OS resources.
* Limited scalability: You can’t just spin up millions of threads — the OS won’t let you.
* Context-switching overhead: More threads = more work for the scheduler.

This works fine for small workloads, but if you’re building a high-throughput service (think: an API serving thousands of requests per second), it quickly becomes a bottleneck.

## Virtual Threads: A Lightweight Model

Virtual threads are Java-managed threads that sit atop a small pool of OS threads. Instead of binding one Java thread directly to one OS thread, the JVM introduces an indirection layer.

Think of it like:

* Platform Thread = expensive hotel room, limited availability.
* Virtual Thread = Airbnb — lightweight, flexible, and you can spin up thousands without breaking the bank.

When a virtual thread blocks on I/O, it doesn’t hog the OS thread. The JVM simply unmounts it and parks it until it’s ready to resume. The OS thread is then free to run another virtual thread.

## CompletableFuture: Declarative Concurrency

Before Loom, CompletableFuture was our main tool for writing async, non-blocking code. It gives you:

* Chaining: `thenApply()`, `thenCompose()` let you declare pipelines.
* Non-blocking async: It can leverage thread pools to avoid blocking the caller.
* Reactive feel: A stepping stone toward reactive programming without fully embracing it.

But it comes with trade-offs:

* Callback hell: Deep chains can become unreadable.
* Debugging complexity: Stack traces aren’t as friendly.
* Steep learning curve: It forces you to think async-first instead of coding naturally.

## Bringing It Together

Virtual threads don’t replace CompletableFuture; they complement it.

* Use virtual threads when you want to write simple, sequential-looking code that scales (e.g., REST endpoints, database queries).
* Use CompletableFuture when you need to orchestrate async pipelines of tasks (e.g., multiple computations combining into one result).

Virtual threads simplify concurrency by removing the cost barrier. CompletableFuture simplifies concurrency by improving expressiveness. Together, they give Java developers two powerful tools depending on the problem space.