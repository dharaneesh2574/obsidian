# GitHub - Online Schema Migrations with gh-ost

## The Core Problem

GitHub needed to change MySQL table structures while production traffic continued. Its earlier trigger-based migrations coupled application writes to extra migration work, creating contention and making it impossible to pause that work completely. Stopping the bulk copy still left triggers executing on every relevant write.

In [Shlomi Noach's August 2016 account](https://github.blog/news-insights/company-news/gh-ost-github-s-online-migration-tool-for-mysql/), GitHub introduced gh-ost to make migrations controllable and testable. This is a study of that published architecture and its public documentation, not a claim about GitHub's entire current database fleet.

## Architecture & Component Design

**Build a replacement.** gh-ost creates a ghost table, applies the desired schema change while it is empty, and incrementally copies existing rows into it. Meanwhile, the original table continues serving application traffic. The [project overview](https://github.com/github/gh-ost) describes replacing the original only after this background preparation.

**Capture ongoing writes.** Instead of installing triggers, gh-ost connects like a replication client and reads row-based binary-log events. It filters changes for the original table and applies them asynchronously to the ghost table: a concrete use of [[Change Data Capture]]. In the documented replica-connected mode, it reads binlogs from a replica but copies rows and writes the ghost table on the primary. This is not a replica promotion.

**Coordinate the two inputs.** The [triggerless design](https://github.com/github/gh-ost/blob/master/doc/triggerless-design.md) queues copy tasks and change-application tasks, then executes writes to the ghost table sequentially through one connection. It also uses a small changelog table for heartbeats and coordination markers. That utility table is distinct from a trigger-populated queue of application changes.

The source-derived lifecycle is:

`prepare ghost table → copy rows + apply binlog changes → catch up under write lock → atomic table swap`

**Control production impact.** Load thresholds, replication lag, or manual instructions can trigger throttling. gh-ost then pauses row copying, change application, and binlog inspection; low-volume heartbeat and status writes continue. Postponing cut-over is different: copying can finish while ongoing changes continue reaching the ghost table. Operators can defer the final switch without deliberately freezing synchronization. See [throttling](https://github.com/github/gh-ost/blob/master/doc/throttle.md) and [cut-over controls](https://github.com/github/gh-ost).

**Switch without a missing-table interval.** An asynchronous replica of the data can still have unapplied changes when copying ends. gh-ost blocks writes, drains remaining changes, then swaps names atomically. Its [cut-over protocol](https://github.com/github/gh-ost/blob/master/doc/cut-over.md) uses separate locking and renaming connections plus a sentry table to prevent a premature rename. A failed attempt returns to the pre-cut-over state so the original remains available and another attempt can follow.

## Trade-offs & Bottlenecks

“Online” does not mean lock-free: requests can briefly wait at cut-over. Asynchronous processing moves correctness and coordination complexity into the migration tool rather than leaving all propagation inside database transactions.

Pausing protects production capacity but accumulates lag. Required binlogs must remain available until processing resumes, echoing the retention constraint in [[Partitioned Log]] without implying gh-ost uses Kafka. **Design implication:** replay throughput must exceed incoming change volume for a growing backlog to shrink; operational control alone cannot guarantee completion under sustained overload.

The [documented prerequisites](https://github.com/github/gh-ost/blob/master/doc/requirements-and-limitations.md) include row-based logs with full row images and a shared primary or unique migration key without NULL values. Foreign keys and existing triggers are unsupported. These are applicability limits, not minor tuning options. A second table also consumes capacity while migration writes and log transfers compete with live traffic.

## Key Takeaway

Decouple long-running preparation from a short, coordinated activation step. CDC makes the replacement catch up; throttling controls its interference; the cut-over protocol prevents activating incomplete data. These are separate responsibilities, and none substitutes for the others.

Sources linked inline; reviewed September 11, 2026.
