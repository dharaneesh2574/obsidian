# Amazon - Retrying Only Failed SQS Records with Lambda

## The Core Problem

A batch improves processing efficiency, but one failed record can make successful work run again. Consider an illustrative batch of invoice notifications: nine deliveries succeed and one fails. Retrying all ten wastes downstream capacity and can repeat user-visible effects.

Lambda's SQS integration separates the batch used for invocation from the records needing retry. This is the public event-source-mapping contract, not Lambda's separate internal asynchronous invocation architecture.

## Architecture & Component Design

**Poll, hide, invoke.** Lambda polls SQS and invokes the function synchronously with a batch. Messages remain in the queue but become temporarily invisible. A successful batch is deleted; by default, a failed batch becomes available again after the visibility timeout. The integration provides at-least-once processing, so [[Idempotency]] remains necessary even when selective retries are enabled. [SQS integration](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html).

**Return record-level outcomes.** Enable `ReportBatchItemFailures` on the event source mapping. The handler catches individual processing failures and returns their message IDs as `itemIdentifier` entries in `batchItemFailures`. Lambda can then retry those records without retrying the successful records solely because their batch contained a failure. An uncaught function exception still fails the whole batch. [Partial batch responses](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html).

**Respect ordering separately.** For FIFO queues, AWS instructs handlers to stop after the first failure and include failed and unprocessed messages in the failure response. Independent per-record success reporting must not let later work bypass failed prerequisites. The same error-handling guide documents this constraint.

**Bound recovery.** Queue visibility must accommodate processing and retries. AWS recommends at least six times the function timeout, plus the batching window when used. A source-queue redrive policy can move repeatedly failing messages to a dead-letter queue. These controls address retry timing and persistent failures; the partial response identifies which records need another attempt. [Queue configuration](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-configure.html).

## Trade-offs & Bottlenecks

With partial responses enabled, Lambda does not reduce polling because of function failures. Design implication: selective retries avoid penalizing healthy records, but continued polling can still stress an unhealthy downstream service. Monitor failure rates and recovery backlog rather than treating batch success rate as sufficient.

Design implication: acknowledging a record must follow its required durable effect. If a handler reports success too early, selective deletion can remove unfinished work; if an effect succeeds but the response is lost, replay can still occur. A per-record response is not a transaction spanning SQS and an external service.

## Key Takeaway

Choose failure granularity independently from throughput granularity. Batch invocations can remain efficient while retry decisions operate on individual records, provided ordering, visibility, and duplicate-effect handling are explicit.

Sources linked inline; reviewed November 22, 2026.
