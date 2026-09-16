---
layout: post
title: "Should We Retry? Modeling Notification Results in Java"
date: 2026-09-15 18:50:00 -0700
categories: programming java
description: "Why an email timeout needs its own result, and how Java sealed types make missing retry decisions visible."
tags: [java, domain-modeling, distributed-systems, reliability]
comments: false
---

## Executive Summary

A failed email attempt does not always justify another attempt. Rate limiting, an invalid recipient, and a timeout carry different information and require different decisions. A timeout after sending the request is especially difficult: the provider may have accepted the email even though our service never received the response.

Model those distinctions explicitly. Java sealed types let us represent each result with the data that belongs to it, while a separate policy decides what to do next. When we introduce an unknown acceptance result, exhaustive switches make missing decisions visible during compilation. The compiler cannot choose a safe retry strategy, but it can show where we have yet to choose one.

## The Reminder That Might Already Be Sent

Imagine a notification service sending an activity reminder through an email provider. The service submits the request, waits for a response, and times out.

The provider might never have accepted the request. It might also have queued the email before the connection failed. Both situations look like a timeout to our service. Retrying could send a duplicate reminder; stopping could lose the reminder entirely.

Before choosing what to do, we need a result model that can preserve what we know. Start with the responses that seem straightforward. For this integration, assume the provider can:

- Accept the email for processing and return a message ID.
- Temporarily reject the request because of rate limiting.
- Permanently reject the request because the recipient address is invalid.

These responses already call for different actions. Yet an integration often compresses them into a result like this:

```java
import java.time.Instant;

record SendResult(
    boolean successful,
    String providerMessageId,
    boolean retryable,
    Instant retryAt,
    String error
) {}
```

Every caller has to interpret the combinations:

```java
if (result.successful()) {
    // Record acceptance.
} else if (result.retryable()) {
    // Schedule another attempt.
} else {
    // Stop retrying.
}
```

The fields look reasonable individually. Together, they allow this:

```java
new SendResult(
    true,
    null,
    true,
    Instant.now(),
    "Invalid recipient"
);
```

The object claims success, requests a retry, and reports an invalid recipient. The caller above records acceptance because it checks `successful` first. A caller that checks `retryable` first could schedule another attempt. The model permits contradictory answers and leaves callers to resolve them.

Constructor validation could reject that combination. It would still leave callers working with fields whose meaning depends on other fields. Adding a timeout would require another convention: does `retryable` mean we know another attempt is appropriate, or merely that we did not receive a response?

## Represent Each Result Directly

Give the three known responses their own types:

```java
import java.time.Instant;

sealed interface SendResult
    permits Accepted, RetryLater, Rejected {}

record Accepted(
    String providerMessageId
) implements SendResult {}

record RetryLater(
    Instant retryAt,
    String reason
) implements SendResult {}

record Rejected(
    String reason
) implements SendResult {}
```

An `Accepted` result carries a provider message ID. A `RetryLater` result carries an earliest retry time and a reason, derived by the integration from the response and its timing rules. A `Rejected` result explains why sending the same request again will not resolve the rejection.

Each result now has one shape. There is no `Accepted` object that also sets a retry flag. Required values still need constructor validation: records alone do not prevent a null message ID or a missing retry time. The integration should also return a non-null result.

The empty interface has a purpose: it names the complete set of send results our application currently understands. The `permits` clause restricts direct implementations, and these record implementations are implicitly final. See Java's [sealed classes and interfaces documentation](https://docs.oracle.com/en/java/javase/21/language/sealed-classes-and-interfaces.html).

The name `Accepted` is deliberate. Acceptance means the provider took responsibility for processing the email. It does not establish that the email reached the recipient. Delivery and later bounces belong to a subsequent part of the notification lifecycle.

## Let a Policy Choose the Next Action

The result records what the integration learned. A separate operation decides what the application should do with it:

```java
enum NextAction {
    RECORD_ACCEPTANCE,
    SCHEDULE_RETRY,
    STOP_RETRYING
}

final class RetryPolicy {

    NextAction decide(SendResult result) {
        return switch (result) {
            case Accepted accepted ->
                NextAction.RECORD_ACCEPTANCE;

            case RetryLater retry ->
                NextAction.SCHEDULE_RETRY;

            case Rejected rejected ->
                NextAction.STOP_RETRYING;
        };
    }
}
```

These examples target Java 21 or later, where pattern matching for `switch` is a standard feature. Each case matches a result type and gives that branch a variable of that type. Because the switch covers the sealed hierarchy, it needs no `default` branch. Java documents this under [type coverage in pattern switches](https://docs.oracle.com/en/java/javase/21/language/pattern-matching-switch.html).

This simplified policy returns a decision. An application service uses that decision and the original result to record the provider message ID, persist another attempt for `retryAt`, or record the rejection. Database writes and scheduling happen there.

In production, the policy also needs context such as the attempt count and the reminder's expiry. `RetryLater` means the provider's response allows another attempt; the application may still stop because its retry budget is exhausted or the reminder is no longer useful.

Other operations can consume the same result. A metrics recorder can count acceptances and rejections. An operational dashboard can explain why a reminder is waiting. Those responsibilities do not need to become methods on `SendResult`.

## The Timeout Reveals a Missing Decision

Return to the request that timed out after submission. None of the three variants describes it accurately. We have no acknowledgement for `Accepted`, no confirmed temporary rejection for `RetryLater`, and no permanent rejection for `Rejected`.

Giving it a retryable flag would erase the uncertainty that matters to the decision. Preserve that uncertainty as a fourth result:

```java
sealed interface SendResult
    permits Accepted, RetryLater, Rejected, AcceptanceUnknown {}

record AcceptanceUnknown(
    String attemptId
) implements SendResult {}
```

Here, `attemptId` identifies a local attempt persisted before contacting the provider. It lets the application find the original request and any correlation information. A local ID alone does not make the provider able to look up or deduplicate a request.

Recompile the application with this revised hierarchy. The earlier `RetryPolicy` switch no longer compiles because it has no branch for `AcceptanceUnknown`. Any other switch that relied on enumerating the three variants without a catch-all must also be revisited.

That is the useful pressure this design creates. The metrics operation needs to decide how to count an unknown result. The dashboard needs to decide how to display it. The retry policy needs to decide how to proceed.

Adding `default -> NextAction.SCHEDULE_RETRY` would make the switch compile, but it would also route future variants into an existing action without requiring a review. For operations that should consider each result explicitly, keep the individual cases and omit a catch-all.

## Make the Recovery Decision Explicit

Suppose this provider supports looking up a submission using a client reference included in the original request. We choose to reconcile unknown attempts before deciding whether to send again. Add `RECONCILE_ACCEPTANCE` to `NextAction` and this branch to the policy's switch:

```java
case AcceptanceUnknown unknown ->
    NextAction.RECONCILE_ACCEPTANCE;
```

The application service can schedule reconciliation using the stored attempt. Reconciliation may itself remain inconclusive; a missing lookup result establishes non-acceptance only if the provider's contract supports that conclusion.

Another provider might support idempotent submission. Retrying can then reuse the original operation's idempotency key and payload within the provider's documented guarantees and retention window. A fresh key on each attempt would defeat that protection. The [Amazon Builders' Library discussion of safe retries](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/) explains why caller-supplied identifiers matter when a response is lost.

If the provider offers neither reliable reconciliation nor idempotent submission, the application must choose how to handle the risk of duplicates versus missed reminders. A new type cannot remove that trade-off. It can keep uncertainty visible until the policy addresses it.

**The compiler cannot decide how to recover. It can make us confront the missing decision.**

## Use This Where You Own the Result Model

This design makes adding an operation straightforward: implement another consumer of `SendResult`. Adding a result variant means reviewing existing operations that enumerate the variants. That review is valuable when a new result changes what the application knows or what it should do next.

A sealed hierarchy fits a service that owns its result model and recompiles its consumers together. It is less suitable for an extension interface meant to accept arbitrary implementations from external plugins. Closing the set of results is a design choice with a cost.

For a notification service, start by naming the outcomes that change the decision. Keep provider acceptance distinct from delivery, and preserve unknown acceptance when the response is lost. Put the recovery decision in an explicit policy, with the provider's guarantees and the reminder's useful lifetime as inputs. That gives the next operational distinction a clear place in the model and makes the code that must respond to it visible.
