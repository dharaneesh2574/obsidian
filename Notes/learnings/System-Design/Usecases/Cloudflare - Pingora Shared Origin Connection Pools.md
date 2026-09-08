# Cloudflare - Pingora Shared Origin Connection Pools

## The Core Problem

When Cloudflare must fetch from an origin server, opening a new connection adds TCP and potentially TLS setup before useful application traffic flows. Its earlier NGINX-based service kept a separate connection pool in each worker process. A request could not use an idle connection owned by another worker; adding workers fragmented reuse further.

This note covers Cloudflare's September 2022 origin-facing proxy design, supplemented by later public framework documentation. It does not assume that today's production configuration matches the open-source examples.

## Architecture & Component Design

Pingora uses asynchronous Rust, Tokio, and work stealing. Threads share origin connections, so [[Connection Pooling]] can avoid repeated setup without per-process isolation. This sharing is within the proxy, not one socket pool spanning Cloudflare's global fleet.

The framework separates policy from transport machinery. Its 2024 open-source introduction shows an application implementing `ProxyHttp`, with `upstream_peer()` selecting an origin and returning an `HttpPeer`. That object describes the destination and connection options, including TLS settings. Filters and callbacks can modify requests or implement application policy, while the framework handles common proxy work such as parsing, connection establishment, and forwarding.

Reuse does not mean any socket can serve any request. The pooling guide requires matching peer properties, including IP and port, scheme, SNI, client certificate, certificate-verification settings, and proxy settings. A failed request makes its connection non-reusable. This compatibility boundary preserves correctness even when sharing would otherwise improve the reuse rate.

The documented request lifecycle can be summarized as:

```text
Read downstream request headers → select upstream peer
  → reuse compatible connection, or establish a new one
  → forward request headers
  → stream request body upstream and response downstream
  → recycle eligible connections after completion
```

Streaming in both directions matters: an origin can respond while the client is still uploading. The proxy does not inherently need to buffer the entire upload before receiving that response. This follows from the documented duplex forwarding phase, not an assumption about every application built on Pingora.

Cloudflare reported a 5 ms improvement in median time-to-first-byte and 80 ms at the 95th percentile in 2022, attributing the gains to better connection reuse. Those are historical deployment measurements, not a benchmark promise for other workloads.

## Trade-offs & Bottlenecks

- **Reuse versus resource retention:** Idle connections still occupy resources on both ends. HTTP/1.1 guidance recommends limiting simultaneous connections to a server. Keeping every connection forever is not a valid capacity strategy.
- **Compatibility versus hit rate:** Distinct TLS identities or connection policies must remain separate. A higher reuse percentage is not useful if it crosses the wrong authentication boundary.
- **Retry versus duplicate effects:** The failover guide distinguishes connection failure before anything was sent from errors during forwarding. It warns against retrying already-sent non-idempotent requests without knowledge of the origin's behavior. [[Idempotency]] is an application guarantee, not a property created by the pool.
- **Committed downstream response:** Once response headers have reached the client, the guide says the proxy cannot transparently replace the result through failover; it must terminate the failed request and record the error.
- **Shared-state cost — design implication:** Wider sharing reduces pool fragmentation but requires coordination. Contention, slow origins, and application callbacks can still limit throughput; the cited sources do not establish one universal pool-size or locking strategy.

## Key Takeaway

Pingora illustrates how ownership boundaries can matter more than faster request-processing code. Sharing reusable connections removes repeated handshake work, while explicit peer identity, protocol completion, and retry rules keep that optimization correct. Measure the whole request path, not just time spent executing proxy code.

Sources: [Cloudflare: Pingora architecture, 2022](https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/) · [Cloudflare: Open-source framework, 2024](https://blog.cloudflare.com/pingora-open-source/) · [Pooling](https://github.com/cloudflare/pingora/blob/main/docs/user_guide/pooling.md) · [Request lifecycle](https://github.com/cloudflare/pingora/blob/main/docs/user_guide/phase.md) · [Failover](https://github.com/cloudflare/pingora/blob/main/docs/user_guide/failover.md) · [RFC 9112 §9](https://www.rfc-editor.org/rfc/rfc9112.html#section-9). Public documentation reviewed September 8, 2026.
