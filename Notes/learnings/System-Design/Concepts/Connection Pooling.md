# Connection Pooling

## What It Is

Connection pooling retains established network connections so later operations can reuse them. A pool manages reusable transport state, not cached application responses: a request still reaches the remote service even when no new connection is opened.

## Why It Exists

Repeated connection setup adds network round trips and consumes CPU, especially when TLS is involved. Reusing connections amortizes that work across requests. HTTP proxies and database clients both use pooling, although their protocols impose different rules on when a connection is safe to reuse.

## How It Works

1. Identify a compatible destination and connection configuration. An HTTP pool may distinguish the address, TLS server name, client credentials, and certificate-verification settings.
2. Obtain a reusable connection, or establish one when none is available and capacity policy permits.
3. Execute the operation and finish the protocol exchange before returning the connection for another operation. For HTTP/1.1, the client must consume the entire response body before reusing that connection for a subsequent request.
4. Discard connections that fail, expire, or cannot remain persistent. Bound retained resources and handle capacity pressure with an explicit policy.

Pool scope matters: process-local pools isolate workers, while a shared pool can make more existing connections available to concurrent work. Sharing also requires coordination. This is separate from [[Parallel Processing]], which concerns executing work concurrently rather than retaining transport resources.

## Trade-offs

- Keeping connections open saves setup work but consumes sockets, memory, and remote-server capacity.
- More connections can reduce HTTP/1.1 head-of-line blocking, but excessive concurrency can overload the destination.
- A server can close an idle connection just as a new request starts. Retry decisions must consider [[Idempotency]] and whether the first attempt might already have taken effect.
- Pooling is not multiplexing: reusing a connection over time does not itself provide concurrent request streams.
- Incorrect compatibility rules can reuse a connection with the wrong security context. Maximizing reuse is subordinate to correctness.

## Related

[[Cloudflare - Pingora Shared Origin Connection Pools]] · [[Parallel Processing]] · [[Idempotency]]

References: [Pingora pooling guide](https://github.com/cloudflare/pingora/blob/main/docs/user_guide/pooling.md) · [Pingora failover guide](https://github.com/cloudflare/pingora/blob/main/docs/user_guide/failover.md) · [HTTP/1.1 connection management, RFC 9112 §9](https://www.rfc-editor.org/rfc/rfc9112.html#section-9). Reviewed September 8, 2026.
