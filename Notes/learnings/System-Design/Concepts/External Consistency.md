# External Consistency

## What It Is

External consistency is a transaction-ordering guarantee in which committed transactions appear to execute one at a time, and that serial order also respects real-world completion order. If transaction A finishes before transaction B begins committing, every observer must see A ordered before B.

It is stronger than ordinary serializability. A serializable database may choose any equivalent serial order for concurrent activity, including an order that conflicts with what clients observed in wall-clock time. External consistency rules out that surprise.

## Why It Exists

Applications often coordinate through events outside the database. A user might see one transaction complete, then trigger another request handled in a different region against unrelated rows. If the database assigns those transactions timestamps using unsynchronized local clocks, a later snapshot could include the second transaction without the first. External consistency makes database history agree with the order users and external systems observed.

## How It Works

The database assigns every transaction a commit timestamp that preserves real-time precedence, then serves reads from snapshots ordered by those timestamps. Google Spanner combines synchronous Paxos replication with TrueTime, an API that returns an interval guaranteed to contain the actual time.

A Spanner leader selects a timestamp at the upper bound of the current TrueTime interval, replicates the write through Paxos, and performs *commit wait*: it does not expose success until TrueTime proves that timestamp is in the past. Multi-version concurrency control then lets a snapshot read select the newest version at or before its read timestamp without blocking writers.

## Trade-offs

The guarantee simplifies application reasoning but demands more infrastructure and coordination. Synchronous replication makes write latency sensitive to quorum placement and network distance. Commit wait adds latency based on clock uncertainty, so clock quality becomes part of database performance. Read-write transactions can hold locks, and transactions spanning partitions also require distributed coordination. Historical versions consume storage, while hot key ranges can still overload their leaders.

## Related

Used in [[Google - Globally Consistent Transactions with Spanner]]. Unlike the eventually consistent designs in [[Amazon - Always-Available Shopping Cart with Dynamo]] and [[Netflix - EVCache Multi-Region Caching]], external consistency preserves a global transaction order even across regions.

External references: [Google Cloud explanation of TrueTime and external consistency](https://docs.cloud.google.com/spanner/docs/true-time-external-consistency) · [Google's original Spanner paper](https://research.google/pubs/spanner-googles-globally-distributed-database-2/)
