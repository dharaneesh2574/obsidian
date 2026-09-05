# Dropbox - Magic Pocket Erasure-Coded Storage

## The Core Problem

Dropbox needed to store a multi-exabyte collection of user file blocks while surviving disk, host, and data-center failures. Several complete replicas would make recovery simple, but the capacity cost would be extreme. Applying [[Erasure Coding]] immediately would save space, yet small incoming writes and degraded reads would become more complicated.

Magic Pocket separates the write-friendly and capacity-efficient parts of the lifecycle. It accepts new data through replication, groups it into immutable units, and erasure-codes those units later. The system must also keep placement metadata consistent during repair and detect latent corruption.

## Architecture & Component Design

A stored block is an opaque file chunk of up to 4 MB that has already been compressed and encrypted. Blocks collect into roughly 1 GB logical **buckets**, and buckets form **volumes**. Open volumes accept writes and are replicated. Closed volumes are immutable and can later be erasure-coded, avoiding parity updates for every small write.

The request path is deliberately separated from background coordination:

- **Frontends** serve reads and writes. A sharded MySQL **Block Index** maps each block hash to its cell, bucket, and checksum.
- A **Replication Table** maps buckets to volumes and records which object-storage devices (OSDs) hold each volume, along with its type and generation.
- **OSDs** persist volume data across failure zones.
- A single **Master** coordinates each storage cell but stays off the data path. It schedules bucket creation, garbage collection, encoding, movement, and repair.
- **Volume Managers**, colocated with OSDs, execute those plans using spare disk and network capacity.

Dropbox lays out a healthy erasure-coded volume so a normal block read can usually touch one spindle. If that fragment is unavailable, a Volume Manager reconstructs the data from surviving OSDs. Dropbox describes a Reed–Solomon-like design related to Local Reconstruction Codes but does not publish its exact production parameters.

Repair is distributed. After an OSD remains unavailable long enough to be treated as failed, the Master closes affected volumes and creates a reconstruction plan. The device shares thousands of volumes with hundreds of other OSDs, so work fans out across many network cards and spindles using [[Parallel Processing]]. Once replacement data is complete, the Master advances the volume generation and updates the Replication Table. The generation fences off stale workers, making the metadata update the ownership commit point.

Durability also requires verification. Independent systems scrub disks, check metadata and extents, inspect deleted data, and verify cross-zone copies, finding corruption before another failure turns it into data loss.

## Trade-offs & Bottlenecks

- Erasure coding reduces steady-state capacity, but encoding, degraded reads, and repair consume CPU, disk I/O, and cross-machine bandwidth.
- Replicated ingestion keeps writes simple and fast, but temporarily costs more space until volumes close and background encoding catches up.
- Immutable volumes simplify movement, checksums, and reconstruction; deletes and garbage collection become asynchronous maintenance work.
- One Master per cell simplifies repair and commit ordering, but Dropbox notes that it limits a cell to roughly 100 PB. Multiple cells provide scale and isolation.
- The Master is not on the request path, yet the Block Index and Replication Table remain critical dependencies for locating data.
- Fast repair shortens the vulnerability window, while excessive repair traffic can hurt foreground latency.

## Key Takeaway

Magic Pocket treats data as replicated while it is small and mutable, then converts closed immutable volumes into erasure-coded layouts. It couples that lifecycle with generation-fenced metadata, parallel repair, failure-domain-aware placement, and continuous verification. The erasure code is only one component: safe ownership transitions and observable repair make it a dependable storage system.

Sources: [Inside the Magic Pocket](https://dropbox.tech/infrastructure/inside-the-magic-pocket) · [Pocket watch](https://dropbox.tech/infrastructure/pocket-watch) · [Disaster readiness: Blackholing an entire data center](https://dropbox.tech/infrastructure/disaster-readiness-test-failover-blackhole-sjc)
