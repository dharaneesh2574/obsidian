# Netflix - EVCache Multi-Region Caching

## The Core Problem

Netflix services repeatedly read recommendations, session-like state, and other expensive-to-fetch data. Serving every request from a database or upstream service would add latency and amplify load. A single cache cluster is not enough: machines disappear, availability zones fail, traffic can move between regions, and cached data is deliberately ephemeral.

EVCache therefore has to keep reads fast during ordinary operation while preserving useful copies across failure boundaries. It does not promise database-style consistency. Netflix explicitly accepts rare write-after-write hazards and failed writes because most entries have short TTLs, then provides repair or application retry options for workloads that need more protection.

## Architecture & Component Design

EVCache exposes a client library backed by groups of Memcached servers in each availability zone. The client discovers current servers through Eureka and uses Ketama [[Consistent Hashing]] to shard keys within a zone. This keeps placement stable when instances are added or replaced and lets aggregate memory and network capacity grow horizontally.

An application-level write is sent to every zonal copy in the region. Reads normally stay in the client's local zone, where Netflix documents sub-millisecond latency for typical 1 KB values; if a local copy is missing or a read fails, the client can retry another copy. This is a deliberate local-read, multi-copy-write pattern: the common path is fast, while a cross-zone fallback trades latency for availability.

Netflix also built Cross-Region Replication (CRR) so cache mutations can follow traffic between AWS regions. Its re:Invent presentation describes a pipeline using Kafka, SQS, load balancers, autoscaling, and EC2 to move cache updates asynchronously. Netflix reported about 30 million cross-region requests per second with P90 latency under two seconds. A 2024 USENIX talk describes the surrounding replication engine, proxy, control plane, and cache warmer.

The two replication layers serve different timescales: synchronous fan-out creates local zonal redundancy for the request path, while asynchronous CRR makes data available in other regions without adding a cross-region round trip to every foreground read. Exact internal message schemas and recovery algorithms are not public in the cited sources and should not be inferred.

## Trade-offs & Bottlenecks

Writing every zonal copy increases write amplification and makes partial success possible. Local reads can then observe different values in different zones until TTL expiry, retry, or consistency repair. Cross-region replication adds further lag, ordering, backlog, and duplicate-delivery concerns; consumers must tolerate an eventually consistent cache.

Fallbacks can also turn a zone problem into load on healthy zones or the source database. EVCache counters this with short timeouts, bounded queues, retries, and fail-fast behavior, but those controls intentionally convert overload into misses or errors. Consistent hashing limits churn when membership changes, yet it cannot solve hot keys, uneven object sizes, or mass cold starts. Memory eviction remains normal, so systems using EVCache as more than an optimization need explicit durability and rebuild strategies.

## Key Takeaway

High availability does not require making every cache read globally consistent. EVCache keeps the foreground path local and fast, replicates synchronously across nearby failure domains, and propagates changes asynchronously across distant ones. The architecture works because its consistency guarantees match the data's ephemeral nature—and because failures are bounded with TTLs, fallback, repair, and fail-fast controls rather than hidden.

Sources: [Netflix EVCache deployment architecture](https://github.com/Netflix/EVCache/wiki) · [Netflix EVCache introduction and operating characteristics](https://netflix.github.io/EVCache/introduction/) · [AWS re:Invent 2023: Netflix multi-region cache replication](https://aws.amazon.com/video/watch/dc20cfe385b/) · [USENIX SREcon 2024: Netflix cache inconsistencies](https://www.usenix.org/conference/srecon24americas/presentation/goel)
