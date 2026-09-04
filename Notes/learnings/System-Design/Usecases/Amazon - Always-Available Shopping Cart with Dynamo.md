# Amazon - Always-Available Shopping Cart with Dynamo

## The Core Problem

Amazon's shopping cart cannot become unavailable just because a disk fails, a network route flaps, or an entire data center is unreachable. Rejecting an "add to cart" action directly harms the customer experience and can lose a sale. At the same time, cart access is structurally simple: fetch or replace a relatively small object by its primary key. It does not require joins or multi-row transactions.

Amazon built the internal Dynamo key-value store around this priority: preserve the ability to read and write during failures, even when replicas cannot immediately agree. The 2007 Dynamo paper reports that the Shopping Cart Service handled tens of millions of requests and more than three million checkouts on a peak day. This system is the original internal Dynamo described in that paper, not the later managed DynamoDB service.

## Architecture & Component Design

Each cart is stored as an opaque value under a key. Dynamo hashes that key onto a ring using [[Consistent Hashing]]. Virtual nodes give each physical server several ring positions, improving load balance and allowing higher-capacity machines to own more ranges. Any node can receive a request, but a coordinator routes it to the key's ordered preference list.

For each key, Dynamo stores `N` replicas on distinct physical nodes, with the preference list spread across data centers. A write succeeds after `W` replicas respond; a read waits for `R`. Services tune `N`, `R`, and `W` for their latency, durability, and consistency needs. Waiting for fewer than all replicas reduces tail latency because the response is no longer gated by the slowest copy.

During a temporary failure, Dynamo uses a *sloppy quorum*: it contacts the first `N` healthy nodes rather than insisting on the key's normal owners. A substitute stores the replica with a hint naming its intended owner. When that owner recovers, *hinted handoff* transfers the data back. This keeps the cart writable through node and network failures.

Concurrent updates can create divergent cart versions. Dynamo attaches vector clocks so it can discard versions that are causally older and identify branches that are genuinely concurrent. If branches cannot be reconciled mechanically, the cart application receives them and merges their contents. The design deliberately moves conflict resolution to reads so writes are not rejected.

Background anti-entropy compares Merkle trees to find differing key ranges without transferring entire datasets. Read repair opportunistically updates stale replicas, while gossip distributes membership and ring-placement changes.

## Trade-offs & Bottlenecks

Availability comes at the cost of eventual consistency. A read may return stale or multiple versions, and application developers must implement correct semantic merging. For carts, preserving additions is usually preferable to losing them, but the paper explicitly notes that deleted items can reappear after reconciliation.

Lower `R` and `W` values reduce latency and tolerate more failures but increase the chance of stale reads or lost durability if several temporary holders fail. Higher values strengthen agreement and durability while making operations wait for more replicas. Because latency is determined by the slowest required response, poor replica placement or a hot key can still damage the 99.9th percentile.

The supporting mechanisms also carry operational costs: hinted replicas consume temporary storage, Merkle trees must be rebuilt when ranges move, vector clocks can grow and require truncation, and gossip produces only an eventually consistent membership view. Dynamo's deliberately narrow interface also excludes multi-key transactions and relational queries, so it fits carts and sessions better than workflows requiring cross-record invariants.

## Key Takeaway

The shopping cart is a case where accepting temporary disagreement is safer than rejecting customer intent. Dynamo combines stable partitioning, tunable replication, failure-aware routing, durable hints, and application-level merging so an item can still be added during infrastructure failure. The architecture succeeds because its consistency model is chosen around the business meaning of the data—not applied as a universal database default.

Sources: [Amazon Science publication page](https://www.amazon.science/publications/dynamo-amazons-highly-available-key-value-store) · [DeCandia et al., *Dynamo: Amazon's Highly Available Key-value Store* (SOSP 2007)](https://cdn.amazon.science/ac/1d/eb50c4064c538c8ac440ce6a1d91/dynamo-amazons-highly-available-key-value-store.pdf)
