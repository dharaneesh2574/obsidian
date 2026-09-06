# LinkedIn - Kafka Event Distribution and Delivery Auditing

## The Core Problem

LinkedIn needed clicks, page views, metrics, and application events to reach many independent systems. Search and stream processing needed fresh data; Hadoop needed complete batches. Direct connections between every producer and destination would make new consumers and recovery increasingly difficult.

This case study covers LinkedIn's published 2015 architecture. Kafka provided a common messaging layer, while surrounding services managed schemas, movement between data centers, and evidence that events arrived. The design let applications share data while operating at different speeds.

## Architecture & Component Design

Kafka organized events into topics backed by a [[Partitioned Log]]. Each partition retained an ordered sequence across reads, allowing multiple subscribers to use the same events. Partitions distributed storage and processing across brokers; within a traditional consumer group, different members handled different partitions using [[Parallel Processing]]. There was no single order across the entire topic.

Consumers fetched batches from an offset and checkpointed their progress. Rewinding that position enabled reprocessing within the retention window. Sequential storage, the operating system's page cache, and batching made persistent messaging practical at high throughput. Batch size traded additional waiting time for fewer I/O operations.

LinkedIn separated queuing, metrics, logging, and tracking into different clusters. Within each category, local clusters received events generated in their data center. MirrorMaker copied these into aggregate clusters that combined events from multiple locations. Forward-only movement prevented replication loops; consumers could obtain the combined feed locally.

The documented flow can be summarized as:

```text
Producers in each data center
            ↓
      Local Kafka clusters
            ↓ MirrorMaker
     Aggregate Kafka clusters
            ↓
  Stream processors · Hadoop · Applications
```

A schema registry supported Avro event encoding and decoding. A REST interface gave non-Java applications access, and a Hadoop bridge published derived results back through Kafka to serving databases. These services made the log usable across teams with different runtimes and data-processing needs.

Delivery required more than healthy brokers. Producers published counts for time intervals to an audit topic. Cluster auditors independently counted observed events, and critical consumers such as Hadoop reported counts at the destination. Comparing the tiers helped engineers locate missing or duplicated traffic along the path. Common producer libraries supplied identifying headers, schema registration, and audit reporting.

## Trade-offs & Bottlenecks

- **Freshness and completeness:** Independent consumers could fall behind. A new event being accepted locally did not mean an aggregate cluster or analytics result already included it. Retention set the maximum replay window.
- **Shared capacity:** LinkedIn identified backlog replay and partition movement as ways to saturate network links and affect other applications. Quotas and safer movement tools were areas of investment in 2015, not completed features assumed here.
- **Recovery semantics:** Processing an event before saving the offset could repeat its effects after a crash. Saving the offset first could skip unfinished work. A durable log alone did not make external database updates exactly once.
- **Audit limits — inference:** Matching aggregate counts cannot prove that every individual event is correct: one missing event and one duplicate could cancel out. Counts are useful operational evidence, but stronger identity checks would be needed for that guarantee.

## Key Takeaway

LinkedIn made a retained event history a shared integration interface. Producers published once, and consumers advanced or replayed at their own pace. The surrounding schema, replication, and audit services mattered as much as the broker: a reliable pipeline must expose where data is delayed or lost across its full path, including the final destination.

Sources: [LinkedIn: Running Kafka at Scale (March 2015)](https://engineering.linkedin.com/kafka/running-kafka-scale) · [LinkedIn: Kafka at LinkedIn (January 2015)](https://www.linkedin.com/blog/engineering/open-source/kafka-linkedin-current-and-future) · [Jay Kreps: The Log](https://www.linkedin.com/blog/engineering/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying) · [Apache Kafka concepts](https://kafka.apache.org/41/getting-started/introduction/) · [Apache Kafka design and delivery semantics](https://kafka.apache.org/41/design/design/). Apache references explain the general mechanisms; later features are not attributed to LinkedIn's 2015 deployment.
