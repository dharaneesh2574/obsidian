# Shuffle Sharding

## What It Is

Shuffle sharding assigns each tenant a small subset of a larger resource pool. Assignments can partially overlap or even coincide. Restricting each tenant's traffic to its assigned resources reduces how widely a noisy or faulty workload can spread.

## Why It Exists

Randomly distributing every request across an entire fleet can let one tenant overload everyone. Fixed, disjoint shards contain that damage, but tenants sharing a shard have the same neighbors. Overlapping subsets provide many more possible assignments without requiring dedicated infrastructure for every tenant.

## How It Works

1. Choose the resource pool and the number of resources per tenant.
2. Create a repeatable assignment using a tenant-derived hash or a stored random selection.
3. Route requests only within that subset. Selection or failover policy determines whether healthy members can absorb work when another member is busy.
4. Monitor capacity and overlap; keep reassignment and recovery from spreading the original overload.

An illustrative queue assignment is:

| Tenant | Eligible queues | Relationship to tenant A |
| --- | --- | --- |
| A | Q1, Q3 | Original workload |
| B | Q2, Q4 | No shared queue |
| C | Q3, Q5 | Shares Q3; Q5 remains an alternative |

If A overloads its queues, C may experience contention on Q3 even though its full assignment differs. This example assumes the queues' processing resources are sufficiently isolated; it is not a production topology.

## Trade-offs

- Partial overlap still permits interference. A low probability of identical subsets is not a guarantee of zero customer impact.
- Larger subsets offer more routing choices but expose more resources to each tenant's load.
- Shared dependencies can defeat queue-level isolation; shuffle sharding is not equivalent to independent, self-contained cells.
- Stateful reassignment needs care: changing where new work goes does not automatically relocate old work.

## Related

[[Consistent Hashing]] emphasizes stable placement during membership changes; shuffle sharding emphasizes limited overlap between tenants. [[Idempotency]] protects against harmful repeated effects during recovery.

Used in [[Amazon - Lambda Asynchronous Invocation Isolation]]. References: [Colm MacCárthaigh: Shuffle Sharding, 2014](https://aws.amazon.com/blogs/architecture/shuffle-sharding-massive-and-magical-fault-isolation/) · [AWS: Shuffle sharding versus cells](https://docs.aws.amazon.com/wellarchitected/latest/reducing-scope-of-impact-with-cell-based-architecture/faq.html). Reviewed September 9, 2026.
