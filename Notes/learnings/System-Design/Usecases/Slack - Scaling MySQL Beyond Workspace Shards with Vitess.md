# Slack - Scaling MySQL Beyond Workspace Shards with Vitess

## The Core Problem

Slack's original database layout kept a workspace's data together on one MySQL shard. That simplified application queries, but adding shards for new workspaces could not solve a single workspace outgrowing the largest available shard. Shard lookup and connection logic also lived inside Slack's web application.

Slack's [December 2020 engineering account](https://slack.engineering/scaling-datastores-at-slack-with-vitess/) describes replacing this model with Vitess. Keyspaces grouped data by growth dimensions such as users, teams, and channels instead of requiring everything to scale with workspace size. The case concerns horizontal placement and query routing, not changing a table's schema in place.

## Architecture & Component Design

**Put routing behind a database interface.** Vitess's [VTGate](https://vitess.io/docs/24.0/concepts/vtgate/) accepts MySQL-protocol connections, routes queries to appropriate VTTablets, and consolidates results. Applications can address a logical database without managing each underlying MySQL host directly. A tablet combines MySQL with its corresponding VTTablet process; the [tablet documentation](https://vitess.io/docs/24.0/concepts/tablet/) distinguishes primary, replica, and read-only roles.

**Manage the database-facing work.** The [Vitess architecture guide](https://vitess.io/docs/24.0/overview/architecture/) identifies VTTablet features including [[Connection Pooling]] and query rewriting. These complement sharding: connection reuse manages access to an individual database, while distributing records across shards changes where storage and query work can be served.

The simplified component path is:

`application → VTGate query routing → selected VTTablet → MySQL`

**Expand a logical keyspace.** Modern [VReplication documentation](https://vitess.io/docs/24.0/reference/vreplication/vreplication/) describes copying source data and then applying ongoing binlog changes to destinations. Streams can filter data for different target shards. This combines initial copying with [[Change Data Capture]], keeping new destinations current while preparation proceeds. Its `VDiff` facility compares consistent source and target snapshots to verify copied data.

The documented [Reshard lifecycle](https://vitess.io/docs/24.0/reference/vreplication/reshard/) separates creating and monitoring the workflow, validating data, switching traffic, and cleaning up. These versioned Vitess references explain the mechanism; they are not evidence that Slack's 2020 migration used today's exact commands or defaults.

**Treat initial adoption separately.** Slack reported building a backfill system with application double-writes and a parallel double-read comparison system to check legacy and Vitess behavior. That historical migration should not be silently rewritten as a VReplication-only workflow. Slack also reported using keyspace splitting to handle a 50% query-rate increase during one week in March 2020.

## Trade-offs & Bottlenecks

Sharding distributes capacity, but a workload can still concentrate on one shard. In [The Query Strikes Again](https://slack.engineering/the-query-strikes-again/), Slack described a bulk user-deletion workload that generated heavy writes, replication lag, and repeated primary failures. Replacement replicas struggled to catch up and were removed by automation that considered them unhealthy, prolonging the failure cycle.

That incident illustrates two distinct constraints: spare hosts do not immediately become caught-up replicas, and aggregate database capacity does not guarantee capacity for a concentrated write burst. Replica reads also have freshness limits because committed primary changes take time to reach replicas.

Resharding itself consumes source resources; Vitess explicitly warns that copy workflows can significantly affect production tablets. **Design implication:** reserve migration headroom and measure replication progress alongside user-facing latency. More shards also add routing and operational work; spreading one query across them is not free [[Parallel Processing]].

## Key Takeaway

Choose a partitioning unit that can grow with the product, and separate logical database access from physical host placement. Vitess gave Slack room to split workloads beyond workspace boundaries, but correctness checks, replication capacity, and hot-shard behavior remained essential parts of the design.

Sources linked inline; reviewed September 14, 2026.
