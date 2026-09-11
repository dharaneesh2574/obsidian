# Change Data Capture

## What It Is

Change data capture (CDC) turns database changes into events that another system can consume. Log-based CDC reads the database's change log instead of asking application code to publish a second write. For example, a MySQL connector can extract row-level inserts, updates, and deletes from the binary log.

## Why It Exists

Copying a table once does not keep a destination current while users continue writing. CDC supplies the changes needed after an initial copy, enabling continuously updated downstream data and live migrations.

## How It Works

A connector establishes a starting dataset and a corresponding log position, then processes later changes. Debezium's MySQL connector uses an initial consistent snapshot followed by binlog streaming. It also tracks schema history so older events are interpreted using the structure that existed when they were recorded.

Different implementations coordinate copying and streaming differently: gh-ost incrementally copies rows while applying captured changes to a replacement table. Do not assume every CDC tool uses Debezium's snapshot algorithm.

## Trade-offs

- Asynchronous capture introduces lag: a committed source write need not already exist downstream.
- Log retention bounds recovery. If required history expires, a consumer cannot simply continue from its old position.
- Replay safety requires deliberate handling. As discussed in [[Idempotency]], repeating a downstream effect is not automatically harmless.
- Schema changes, row identity, and copy/stream boundaries are correctness concerns, not just transport details.

## Related

[[Partitioned Log]] describes a possible downstream event transport; CDC describes where change events originate. MySQL's binlog is not itself a Kafka partitioned topic. [[GitHub - Online Schema Migrations with gh-ost]] applies captured changes to a replacement table rather than an analytics pipeline.

References: [Debezium MySQL connector](https://debezium.io/documentation/reference/stable/connectors/mysql.html) · [gh-ost triggerless design](https://github.com/github/gh-ost/blob/master/doc/triggerless-design.md) · [gh-ost retention and throttling](https://github.com/github/gh-ost/blob/master/doc/throttle.md). Reviewed September 11, 2026.
