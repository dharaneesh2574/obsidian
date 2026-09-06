# Google - Dapper Distributed Tracing

## The Core Problem

Google services span many software modules, languages, machines, and data centers. A single user request may fan out through several RPC layers, so a slow response cannot be explained reliably from one process’s logs. Asking every application team to maintain custom tracing would create uneven coverage and too much production overhead.

Dapper was designed as always-on infrastructure with three goals: low overhead, application-level transparency, and broad deployment. It needed to reconstruct a request’s causal path without making the tracing pipeline part of that request’s availability or latency path.

## Architecture & Component Design

Dapper represents each request as a tree of **spans**. Every span records a human-readable operation name, start and end events, RPC timing, optional annotations, a span ID, and its parent’s span ID. All spans for the same request share a trace ID. Parentage reveals causality even when [[Parallel Processing]] makes sibling calls overlap.

Instead of instrumenting every application, Google inserted tracing into a few common infrastructure libraries. The shared RPC framework creates spans and propagates trace and span identifiers from client to server. Threading and control-flow libraries carry the same context into callbacks and asynchronous work. Applications can add annotations, but limits prevent those annotations from displacing structural trace data. This approach made [[Distributed Tracing]] available to largely unchanged services and across C++ and Java boundaries.

Tracing is sampled to protect foreground performance. A root request makes the initial sampling decision, which travels with its trace context so participating services either record the trace together or omit it together. The paper says Dapper’s first production deployment sampled one in 1,024 requests by default, while lower-volume applications could use higher rates. This is a documented historical configuration, not a claim about Google’s current sampling policy.

Collection happens out of band. Processes write sampled spans to local log files; host daemons pull them into regional collectors, which store them in Bigtable. A trace occupies one sparse row and its spans occupy columns, accommodating traces whose width and depth vary substantially. Keeping collection off the request path avoids adding trace payloads to application responses and supports asynchronous work that is not perfectly nested.

Dapper’s query API can retrieve a known trace, scan data in bulk, or use an index based on service, host, and time. A web interface reconstructs span trees for engineers. Because even sampled production traffic generated heavy write volume, the collection layer applied a second adjustable sampling stage. It hashes the shared trace ID, retaining or discarding the complete trace consistently before Bigtable storage.

## Trade-offs & Bottlenecks

- Shared-library instrumentation delivers broad coverage cheaply, but custom communication paths remain blind until instrumented. Updating code linked into many applications is operationally difficult.
- Sampling keeps overhead low but can miss rare problems. Fixed rates also give busy services far more traces than quiet ones, motivating adaptive policies.
- A second collection-stage sampler controls Bigtable cost and write throughput, but discarding a trace after hosts recorded it spends some local work without producing a queryable result.
- Out-of-band collection isolates production requests, yet traces appear later and can be incomplete while spans arrive from different hosts.
- Cross-machine clock skew can distort timing. Dapper’s causal IDs still establish relationships, but timestamps need careful interpretation.
- Custom annotations improve diagnosis while creating volume and sensitive-data risks; size limits and data-governance rules are necessary.

## Key Takeaway

Dapper scaled tracing by standardizing the narrowest shared layer: request context in common RPC and concurrency libraries. Trace trees made distributed work understandable, coherent sampling bounded cost, and asynchronous collection kept observability failures away from user traffic. The general lesson is that tracing becomes dependable infrastructure when context propagation is ubiquitous and cheap—not when individual services merely emit more logs.

Sources: [Google Research: Dapper, a Large-Scale Distributed Systems Tracing Infrastructure](https://research.google/pubs/dapper-a-large-scale-distributed-systems-tracing-infrastructure/) · [Original Google technical report PDF](https://research.google.com/archive/papers/dapper-2010-1.pdf) · [W3C Trace Context](https://www.w3.org/TR/trace-context/)
