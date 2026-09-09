# Amazon - Lambda Asynchronous Invocation Isolation

## The Core Problem

An asynchronous compute service must absorb bursts without letting one customer's backlog delay every other customer. More queues provide capacity, but distributing each tenant's requests across all of them also distributes that tenant's overload.

AWS's March 2025 engineering account describes how Lambda evolved its internal asynchronous invocation system toward tenant-aware routing. This case study covers that internal path and the documented public delivery contract, not a claimed production queue count or universal shard size.

## Architecture & Component Design

A caller sets `InvocationType` to `Event`. Lambda queues the event and returns HTTP `202`; this acknowledges acceptance, not successful execution. A separate poller reads queued work and invokes the function synchronously. The caller no longer waits for the function result, and Lambda manages subsequent execution attempts.

The basic separation is:

```text
Caller → Lambda intake → internal queue → poller → function
Caller ← HTTP 202 after acceptance
```

The routing strategy determines which workloads share waiting time. AWS describes an intermediate design using [[Consistent Hashing]] to keep each tenant on one queue. That contained a tenant's impact, but customers assigned to the same queue still shared its backlog.

Lambda then applied [[Shuffle Sharding]]: assign a tenant several queues and put each new event into the assigned queue with the smallest backlog. This combines a restricted set of neighbors with a choice among available paths. The article also describes monitoring queue depth and moving spiky tenants to dedicated queues. Exact selection parameters and reassignment protocols are not published there.

The general isolation principle is limited overlap, not exclusive ownership. Two tenants can have different queue sets and still share one congested queue. Choosing another eligible queue can help new arrivals; it does not establish that an event already waiting elsewhere is automatically moved. That last distinction is a design implication, not a documented Lambda migration algorithm.

Public metrics expose the difference between intake and progress:

- `AsyncEventsReceived` counts events successfully queued.
- `AsyncEventAge` measures time from queueing to invocation; retries and throttling can increase it.
- `AsyncEventsDropped` counts events discarded without successful function execution.

Read these alongside `Errors` and `Throttles`. A healthy acceptance rate can coexist with deteriorating processing latency, so checking only the intake endpoint would miss the operational problem.

## Trade-offs & Bottlenecks

- **Isolation versus capacity:** Shuffle sharding cannot create execution capacity. Lambda throttles when available concurrency is insufficient. Separating tenants' queues does not eliminate function-level or account-level limits.
- **Buffering versus deadlines:** Backlogs absorb temporary mismatches, but events have a bounded lifetime. Lambda discards events that expire or exhaust processing attempts. Configure a dead-letter queue or on-failure destination when discarded work needs investigation or recovery.
- **Retries versus repeated effects:** By default, function errors receive two additional attempts. Throttling and system errors follow a different retry policy, lasting up to six hours by default. Duplicate delivery can also occur without a function error; handlers need [[Idempotency]] where repeated side effects would be harmful.
- **Logical versus physical isolation:** The general shuffle-sharding model does not prove that every queue has independent downstream resources. A shared bottleneck can still affect many subsets. This is an architectural limitation, not evidence of a particular Lambda incident.
- **Capture versus observability:** Configuring a failure destination is not proof that delivery there succeeded. `DestinationDeliveryFailures` and `DeadLetterErrors` expose failures in those recovery paths.

## Key Takeaway

Lambda separates accepting work from executing it, then limits tenant interference through placement choices. Shuffle sharding narrows who can be affected; queue-age metrics reveal waiting; bounded retries and idempotent handlers govern recovery. Reliable asynchronous processing requires all three concerns—capacity, isolation, and delivery semantics—to remain explicit.

Sources: [Anton Aleksandrov and Rajesh Kumar Pandey: Lambda engineering account, March 17, 2025](https://aws.amazon.com/blogs/compute/handling-billions-of-invocations-best-practices-from-aws-lambda/) · [Asynchronous invocation](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async.html) · [Errors and retries](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async-error-handling.html) · [Lambda metrics](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-metrics-types.html) · [AWS shuffle-sharding model](https://aws.amazon.com/blogs/architecture/shuffle-sharding-massive-and-magical-fault-isolation/) · [Isolation boundaries](https://docs.aws.amazon.com/wellarchitected/latest/reducing-scope-of-impact-with-cell-based-architecture/faq.html). Documentation reviewed September 9, 2026.
