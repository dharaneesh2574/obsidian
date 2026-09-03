# Consistent Hashing

## What It Is

Consistent hashing maps both keys and storage nodes into the same logical hash space. A key belongs to the next node at or after its position, often visualized as moving clockwise around a ring. When a node joins or leaves, only a portion of the keys should move instead of almost every key being remapped.

## Why It Exists

A simple rule such as `hash(key) mod N` works while the number of nodes stays fixed. Changing `N` changes the result for most keys, which can trigger a large cache miss storm or an expensive data migration. Distributed caches and key-value stores need a placement rule that can scale the fleet or replace failed machines with less disruption.

## How It Works

The system hashes each physical node to positions in the key space and hashes every key into that same space. Lookup walks from the key's position to the next node. Production implementations commonly give each physical node many virtual positions so uneven random placement does not overload a few machines.

Adding a node transfers only the ranges immediately before its positions; removing one transfers those ranges to their next owners. Clients or routing services must still agree on the current membership and hash function. Netflix's EVCache uses the Ketama variant inside each availability-zone cache group, while service discovery tells clients when server membership changes.

## Trade-offs

Consistent hashing reduces remapping, but it does not eliminate movement or guarantee perfect balance. Virtual nodes improve balance at the cost of a larger routing table. Heterogeneous machines need weighted placement, and a single popular key can remain hot regardless of how evenly the total keyspace is divided. Stale membership views can also route different clients inconsistently during topology changes, so discovery and rollout behavior are part of the design.

## Related

[[Parallel Processing]] also distributes work across machines, but consistent hashing specializes in stable ownership rather than executing one job concurrently.

Used in [[Netflix - EVCache Multi-Region Caching]]. External references: [Karger et al.'s original consistent-hashing paper](https://people.csail.mit.edu/karger/Papers/web.pdf) · [Netflix EVCache deployment documentation](https://github.com/Netflix/EVCache/wiki)
