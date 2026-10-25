# Amazon - Distributing Hot Writes with DynamoDB Write Sharding

## The Core Problem

A table can have enough aggregate capacity while incoming writes concentrate on one partition key. An illustrative example is an order-ingestion service grouping every new order under today's date. The convenient grouping also becomes a concentration point.

Amazon's [partition-key guidance](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html) recommends distributing activity across keys in both tables and secondary indexes. Physical partitions have throughput limits, and larger items consume more capacity per operation. This note studies AWS's documented DynamoDB application design pattern, not an undocumented Amazon retail deployment or the original Dynamo shopping-cart architecture.

## Architecture & Component Design

**Expand the logical key space.** AWS documents [write sharding](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-sharding.html) by appending suffixes to a common key. Instead of writing everything under one date, write under `date.1` through `date.N`. These are logical keys, not promises of one dedicated physical partition per suffix.

**Choose random or reproducible routing.** A random suffix distributes writes, but a reader cannot derive the chosen suffix from an order ID alone. A calculated suffix based on the order ID makes the destination reproducible. Given the date, ID, and agreed calculation, the application can reconstruct the partition key for an individual lookup. AWS illustrates both strategies; its example shard count is not a universal setting.

**Reassemble group reads.** Retrieving an entire day's orders still requires querying every suffix and combining results. As an application design implication, bounded [[Parallel Processing]] can overlap these independent requests; it does not remove their total work.

**Track progress per query.** DynamoDB's [Query API](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Query.html) accepts one partition-key value. Each response is limited by the evaluated-item limit or 1 MB of data, before filtering. Continue a shard's query by passing its `LastEvaluatedKey` as `ExclusiveStartKey`. A filtered page can be empty yet still contain a continuation key, so zero returned items does not mean that shard is finished. Design implication: the coordinator needs separate pagination state for each suffix.

## Trade-offs & Bottlenecks

The read interface becomes more complicated: per-query sort-key ordering is not an automatically merged ordering across all suffixes. Applications needing a combined ordered result must assemble it themselves. A `FilterExpression` also does not reduce consumed read capacity: filtering happens after items are read, according to the Query contract.

Sharding does not remove table-level throughput limits. AWS's partition-key guidance explicitly distinguishes per-partition capacity from aggregate provisioned or on-demand limits. Distribution and sufficient capacity are separate requirements.

Design implication: choose the suffix count together with the access patterns. Extra destinations add group-read coordination, while a poor calculated distribution can leave traffic concentrated. Validate both ingestion and reconstruction, rather than optimizing writes alone.

## Key Takeaway

Write sharding exchanges a simple grouping key for distributed write destinations. The corresponding read design—routing, fan-out, pagination, and result assembly—is part of the architecture, not an afterthought.

Sources linked inline; reviewed October 25, 2026.
