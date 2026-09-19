# Google - PubSub Subscriber Flow Control and Lease Management

## The Core Problem

A message service can deliver work faster than a subscriber can finish it. Pulling more messages does not create more processing capacity: unfinished work accumulates, consumes resources, and may exceed acknowledgment deadlines. Redelivery can then add work precisely when the subscriber is already struggling.

Google Cloud Pub/Sub separates three controls: how much unfinished work a client accepts, how much it processes concurrently, and how long delivery remains outstanding before recovery. This case covers documented pull-subscriber behavior with high-level client libraries and default at-least-once delivery, not undocumented broker internals or exactly-once mode.

## Architecture & Component Design

**Bound outstanding work.** Subscribers configure limits for outstanding message count and bytes. These describe delivered messages not yet acknowledged or negatively acknowledged, rather than a fixed messages-per-second allowance. When either limit is exceeded, the client stops pulling more until outstanding work is acknowledged or negatively acknowledged. Google documents these controls in its [subscriber best practices](https://docs.cloud.google.com/pubsub/docs/subscribe-best-practices).

**Let the service retain the excess.** The [flow-control guide](https://docs.cloud.google.com/pubsub/docs/flow-control) positions this mechanism as protection against transient spikes, buying time to process the backlog or add capacity. Limits belong to subscriber clients; they are not a single global setting shared by all subscribers. Design implication: size the admitted workload for the resources of each running instance, including message-size variation, rather than treating a count limit as a complete memory budget.

**Tune execution separately.** [[Parallel Processing]] depends on streams, threads, and available processing resources—not simply on accepting more messages. Google's [concurrency guide](https://docs.cloud.google.com/pubsub/docs/concurrency-control) distinguishes pull streams, message-processing executors, and lease-management executors in the Java client. Other language libraries expose different settings. Increasing concurrency may help when processing is not CPU-bound; excessive threads are not a universal throughput improvement.

**Renew deadlines while work continues.** Lease management extends acknowledgment deadlines for messages that are still being processed. High-level libraries expose a maximum total extension period and bounds on individual extensions. The total period limits how long the client keeps extending a message; an individual extension affects how long recovery may wait if that client subsequently crashes. These are distinct settings in the [lease-management documentation](https://docs.cloud.google.com/pubsub/docs/lease-management).

An illustrative lifecycle is:

`receive within flow limits → process while extending deadline → acknowledge completed work → admit more`

If processing does not complete and deadline extensions stop, the message can become eligible for redelivery. This is recovery behavior, not evidence that the first attempt had no side effects.

## Trade-offs & Bottlenecks

- **Protection versus throughput:** tight limits can leave useful capacity idle; loose limits can accumulate excessive unfinished work. Design implication: tune against actual processing latency and downstream capacity, not delivery speed alone.
- **Patience versus recovery:** short deadlines increase the risk of premature redelivery; long extensions can delay another subscriber taking over failed work. Lease renewal accommodates variable processing times without choosing one large deadline for every message.
- **Duplicates remain possible:** default Pub/Sub delivery is at least once, and acknowledged messages can occasionally reappear. The documented deadline is not an absolute no-redelivery guarantee in this mode. [[Idempotency]] is therefore still relevant to the business effect, not just the acknowledgment call. [Delivery behavior](https://docs.cloud.google.com/pubsub/docs/subscribe-best-practices).
- **Design implication: backpressure is not capacity.** Flow control limits admission; it cannot make a permanently undersized worker pool catch up. Sustained overload requires more effective processing capacity, less incoming work, or a different processing design.

## Key Takeaway

Keep admission, execution, and recovery separate. Pub/Sub's flow limits bound accepted work, concurrency settings control processing, and renewable deadlines accommodate slow handlers. A reliable subscriber needs all three plus duplicate-safe effects; extending deadlines alone only postpones the consequences of overload.

Sources linked inline; reviewed September 19, 2026. The lifecycle and labeled design implications are explanatory synthesis.
