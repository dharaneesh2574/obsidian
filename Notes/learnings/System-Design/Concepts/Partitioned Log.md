# Partitioned Log

## What It Is

A partitioned log stores an event stream as several ordered sequences. New records are appended to one partition and receive a position called an offset. Ordering is defined within that partition; offsets from different partitions do not establish a global order.

## Why It Exists

Many applications need the same events at different speeds. A search index may update continuously while an analytics job reads in batches. Retaining a shared history lets each subscriber advance independently, recover from an interruption, or replay retained events after a processing bug is fixed. Producers do not need a separate integration with every destination.

## How It Works

The producer chooses a partition, often using an entity key so related events share an ordered sequence. The storage service appends records and can replicate each partition for durability. Reading an event does not delete it; retention policy determines which history remains available.

Each subscriber tracks its position in each partition. In Kafka's traditional consumer-group model, one group member owns a partition at a time, while other members handle different partitions through [[Parallel Processing]]. Independent groups can read the same records for different purposes.

A saved offset is a recovery checkpoint, not proof that an external side effect happened exactly once. If a worker updates a database and crashes before saving its offset, it can repeat that update after restarting. Idempotent writes or atomic coordination between output and offset are needed when duplicates matter.

## Trade-offs

- Partitioning increases throughput but limits ordering to each partition; popular keys can create uneven load.
- Retention consumes storage. Replay cannot recover history that has expired, so rebuilding may also require a snapshot.
- Consumers progress independently, making lag an explicit part of data freshness.
- Replaying a large backlog competes with live traffic for disk and network capacity.

## Related

- [[Parallel Processing]] — partitions provide independent units of work.
- [[LinkedIn - Kafka Event Distribution and Delivery Auditing]] — a shared log combined with replication between clusters and delivery checks.
- [Jay Kreps: The Log](https://www.linkedin.com/blog/engineering/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)
- [Apache Kafka concepts](https://kafka.apache.org/41/getting-started/introduction/)
- [Apache Kafka consumer position and delivery semantics](https://kafka.apache.org/41/design/design/)
