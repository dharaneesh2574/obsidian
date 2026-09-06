# Distributed Tracing

## What It Is

Distributed tracing records how one logical request moves through services, processes, threads, queues, and machines. The request becomes a **trace**, while each unit of work becomes a **span** containing an operation name, timing, attributes, and status. Trace and parent identifiers connect those spans into a causal graph.

Unlike an ordinary log search, a trace preserves the relationship between upstream and downstream work. It can show that a slow page was waiting on one particular database call even when thousands of unrelated requests produced logs at the same time.

## Why It Exists

In a distributed system, end-to-end latency and failure are emergent properties. A frontend can call several services, which call caches, databases, and queues maintained by different teams. Local metrics reveal that a component is slow, but not necessarily which user requests it affected or what caused the delay.

Distributed tracing restores request-level context. Operators use it to locate critical paths, explain tail latency, discover service dependencies, compare deployments, and distinguish local processing time from downstream waiting.

## How It Works

When a request begins, instrumentation creates a trace identifier and a root span. Before a service calls another component, it creates a child span and propagates trace context—at minimum the trace and current parent identifiers—with the request. The receiver extracts that context, records its own child span, and repeats the process.

Completed spans are exported to a collection pipeline and joined by trace identifier in a tracing backend. A query or visualization reconstructs the request graph and its timing. Instrumenting shared HTTP, RPC, database, messaging, and concurrency libraries provides broad coverage without requiring every application team to add tracing logic manually.

Because recording every request is expensive, systems sample traces. A consistent decision should preserve an entire trace rather than keeping unrelated individual spans. Head sampling decides when the trace starts and is cheap; later or tail-aware selection can retain traces because they are slow or erroneous, but needs buffering and more coordination.

## Trade-offs

- **Visibility versus overhead:** More spans and attributes improve diagnosis but increase CPU, network, and storage use.
- **Coverage versus sampling:** Sampling controls cost but can omit rare failures or distort aggregate conclusions.
- **Transparency versus blind spots:** Shared-library instrumentation scales well, while custom protocols and asynchronous boundaries still require correct context propagation.
- **Detail versus safety:** Attributes can accidentally capture credentials, personal data, or high-cardinality values and need strict controls.
- **Timing versus clock quality:** Cross-host timelines are affected by clock skew; causal parentage is often more trustworthy than comparing raw timestamps.
- **Observability versus dependency:** Collection and storage must fail without breaking the production request path.

## Related

- [[Parallel Processing]] — a trace makes concurrent fan-out and its critical path visible.
- [[Google - Dapper Distributed Tracing]] — Google’s shared-library, sampled tracing architecture.
- [Google Research: Dapper paper](https://research.google/pubs/dapper-a-large-scale-distributed-systems-tracing-infrastructure/)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [OpenTelemetry context propagation](https://opentelemetry.io/docs/concepts/context-propagation/)
