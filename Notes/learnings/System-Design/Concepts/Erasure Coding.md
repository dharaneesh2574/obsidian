# Erasure Coding

## What It Is

Erasure coding is a durability technique that transforms data into a larger set of fragments containing both original data and parity. If some fragments disappear, the original data can still be reconstructed from a sufficient subset of the survivors.

Unlike full replication, which stores several complete copies, erasure coding usually provides comparable tolerance for independent failures with substantially less storage overhead. Reed–Solomon codes and related schemes are common examples.

## Why It Exists

Keeping multiple full copies is simple and makes reads cheap, but its capacity cost becomes enormous at petabyte or exabyte scale. Erasure coding trades additional computation and recovery traffic for better storage efficiency while preserving a chosen failure tolerance.

It is most valuable for large, relatively cold, or immutable objects. Such data can be encoded in batches, distributed across disks or failure domains, and reconstructed when hardware fails without paying for several full replicas indefinitely.

## How It Works

1. Split an object or storage extent into data fragments.
2. Compute additional parity fragments from the data fragments.
3. Place the fragments on independent disks, servers, zones, or other failure domains.
4. Serve healthy reads from the fragment layout chosen by the system.
5. When fragments are unavailable, fetch enough surviving fragments and decode the missing data.
6. Rebuild lost fragments onto healthy hardware to restore the intended redundancy.

Real systems often aggregate small objects before encoding them. This amortizes metadata and coding overhead and makes sequential disk access more efficient. They also spread reconstruction across many machines so that repair becomes a [[Parallel Processing]] problem rather than a single-server bottleneck.

## Trade-offs

- **Capacity versus compute:** Erasure coding saves storage but consumes CPU for encoding and decoding.
- **Healthy versus degraded reads:** A carefully designed layout can keep normal reads cheap, while a missing fragment causes read and network amplification.
- **Fast repair versus production traffic:** Aggressive reconstruction reduces the vulnerability window but competes with user requests for disks and bandwidth.
- **Independent versus correlated failures:** The durability model only holds when placement keeps fragments out of the same likely failure domain.
- **Efficiency versus mutability:** Encoding immutable batches is straightforward; frequent small updates may require rewriting parity or using a replication-first staging layer.
- **Data versus metadata durability:** Fragments are useless if the system loses the mappings, generations, or checksums required to find and validate them.

## Related

- [[Parallel Processing]] — distributes reconstruction work across independent disks and hosts.
- [[Dropbox - Magic Pocket Erasure-Coded Storage]] — combines replicated ingestion with erasure-coded immutable volumes.
- [Inside the Magic Pocket](https://dropbox.tech/infrastructure/inside-the-magic-pocket)
- [Pocket watch: How we built a multi-exabyte data verification platform](https://dropbox.tech/infrastructure/pocket-watch)
