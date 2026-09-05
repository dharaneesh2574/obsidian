# Google - Globally Consistent Transactions with Spanner

## The Core Problem

Google needed a database that could place data across machines, data centers, and continents without making application developers reason about conflicting histories. Traditional distributed stores often gained availability and speed by weakening consistency. That is a poor fit when a transaction completes in one region and a later action elsewhere must observe it in the same order.

Spanner was built as a globally distributed, synchronously replicated, multi-version database with externally consistent transactions. [[External Consistency]] means the database behaves as though transactions ran in one serial sequence, and that sequence respects real time: if transaction A finishes before transaction B starts committing, B receives a later timestamp. This lets globally distributed applications reason much like they are using a single-machine relational database.

## Architecture & Component Design

Spanner partitions tables into contiguous key ranges called *splits*. Each split is replicated across distinct failure domains as an independent Paxos group. A leader coordinates writes, and a majority of voting replicas must durably accept a mutation before the group can progress. Transactions spanning multiple splits add two-phase commit across their Paxos leaders; single-split writes avoid that extra protocol.

Paxos orders writes within a group, but separate groups also need timestamps that preserve causality across the whole database. Google's TrueTime service supplies an interval `[earliest, latest]` guaranteed to contain real time. It is backed by redundant GPS and atomic-clock time sources with different failure modes, while daemons on database servers synchronize against those sources.

For a write transaction, the leader chooses a commit timestamp using TrueTime's upper bound, replicates the write through Paxos, and waits until TrueTime's lower bound has passed that timestamp before reporting success. This *commit wait* ensures that once a client observes a commit, every later transaction can be assigned a greater timestamp. Google overlaps much of this wait with replication work; the remaining delay grows with clock uncertainty.

Spanner stores immutable, timestamped versions through multi-version concurrency control. A read-only transaction chooses a timestamp and reads the latest version at or before it, producing a consistent database-wide snapshot without taking read locks or blocking concurrent writes. Strong reads can use the current safe time; explicitly stale reads can often be served closer to the caller with lower coordination cost.

## Trade-offs & Bottlenecks

Global consistency is not free. A write must reach a Paxos majority, so replica geography directly affects latency and failure tolerance. Multi-split read-write transactions add two-phase commit, locking, and more opportunities for contention or aborts. Poor primary-key design can create a hot split whose leader becomes the throughput bottleneck even though the overall database has ample capacity.

TrueTime turns bounded clock uncertainty into an explicit latency cost. If time synchronization degrades, the uncertainty interval widens and commit wait must become longer to preserve correctness. The design also depends on specialized, redundant time infrastructure that ordinary deployments may not possess.

MVCC makes lock-free snapshots possible but retains multiple versions and therefore consumes storage until old versions are collected. Stale reads improve locality and throughput only when the application can intentionally accept an older snapshot. Strong reads and writes still need replicas to prove that they are sufficiently up to date.

Exact production topology, split sizes, and configuration choices vary by workload; the cited sources document the mechanisms rather than one universal deployment shape.

## Key Takeaway

Spanner separates two distributed-systems problems and solves both: Paxos establishes durable agreement within each replicated data range, while TrueTime gives independent ranges a shared, uncertainty-aware timeline. Commit wait converts that uncertainty into a small, measurable delay, and MVCC turns the resulting timestamps into non-blocking consistent snapshots. The result is global scale without forcing every application to repair contradictory transaction histories.

Sources: [Google Research: original Spanner paper](https://research.google/pubs/spanner-googles-globally-distributed-database-2/) · [Google Cloud: life of Spanner reads and writes](https://docs.cloud.google.com/spanner/docs/whitepapers/life-of-reads-and-writes) · [Google Cloud: TrueTime and external consistency](https://docs.cloud.google.com/spanner/docs/true-time-external-consistency) · [Google Cloud engineering explanation of Paxos, TrueTime, and commit wait](https://cloud.google.com/blog/products/databases/strict-serializability-and-external-consistency-in-spanner)
