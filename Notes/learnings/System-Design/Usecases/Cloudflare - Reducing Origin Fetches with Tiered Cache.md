# Cloudflare - Reducing Origin Fetches with Tiered Cache

## The Core Problem

A geographically distributed CDN can still send redundant work to an origin. With independent edge caches, the first request for an image in each location can miss locally and fetch the same bytes again. Cloudflare's Tiered Cache addresses this by making selected data centers serve other caches, rather than letting every local miss reach the origin. [CDN reference architecture](https://developers.cloudflare.com/reference-architecture/architectures/cdn/).

This differs from [[Connection Pooling]]: pooling reduces the cost of establishing connections; a cache hit can eliminate an origin request altogether. The two techniques address different parts of the request cost.

## Architecture & Component Design

For eligible content, the documented request flow is:

1. A client reaches a lower-tier data center. A local hit is served there.
2. A local miss goes to its upper tier. If that tier has the object, it returns a copy without contacting the origin.
3. On an upper-tier miss, the upper tier fetches from the origin, caches the response, and returns it to the lower tier, which also caches it.
4. A later request at another cold lower tier can reuse the warmed upper tier.

This summarizes the reference architecture's two-client traffic-flow diagram: sharing the upper cache removes repeated origin fills across locations. [Request-flow diagram and explanation](https://developers.cloudflare.com/reference-architecture/architectures/cdn/#traffic-flow-tiered-cache-smart-tiered-cache-topology).

Smart Tiered Cache uses performance and routing measurements to choose a low-latency upper tier for an origin. Selection is not simply geographical proximity. With Cloudflare Load Balancing, the documented selection boundary becomes the whole pool, rather than each individual origin. [Smart Tiered Cache](https://developers.cloudflare.com/smart-shield/configuration/smart-tiered-cache/).

Public-cloud networking complicates measurement. An anycast address may answer through different nearby cloud front ends while the actual backend sits elsewhere. A short probe round trip therefore need not identify the backend location; selecting the wrong upper tier can create cross-continent detours. Cloudflare's engineering account describes this measurement problem explicitly. [Public-cloud origin engineering explanation](https://blog.cloudflare.com/smart-tiered-cache-for-public-clouds/).

For supported public clouds, an operator can supply a provider-and-region hint. Cloudflare then selects a primary upper tier near that region and a fallback elsewhere. This is documented topology selection, not a claim that every deployment has one permanent global cache. [Cloud-region hints](https://developers.cloudflare.com/smart-shield/configuration/smart-tiered-cache/#public-cloud-origins).

## Trade-offs & Bottlenecks

- **Origin protection versus miss latency:** Concentrating fills improves reuse, but a distant upper tier adds travel time for lower-tier misses. Cloudflare documents an optional regional tier between the lower and upper tiers to shorten some of those trips. [Topology trade-offs](https://developers.cloudflare.com/reference-architecture/architectures/cdn/#regional-tiered-cache).
- **Topology changes mean warming:** Changing origin IPs or DNS can change assigned upper tiers and temporarily increase misses while their caches refill. [Operational caveats](https://developers.cloudflare.com/smart-shield/configuration/smart-tiered-cache/#caveats).
- **Eligibility comes first:** Cloudflare does not cache HTML or JSON by default. Default behavior also excludes non-GET requests and responses with conditions such as `Cache-Control: private` or `no-store`; configuration can alter caching behavior. Adding tiers cannot make private content safe to share. [Default cache behavior](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/).
- **No global single-fetch promise:** Cloudflare documents request collapsing for simultaneous misses on the same asset within one data center. That scope does not establish exactly one origin fetch worldwide across failures, expiration, or topology changes. [Request collapsing](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/#request-collapsing).

## Key Takeaway

Design implication: scale the number of client-facing caches without making origin load grow proportionally with cold locations. Shared upper tiers improve reuse, but topology, cache eligibility, and miss-path latency remain separate decisions. Measure origin traffic and user latency together rather than treating a higher cache-hit ratio as the whole objective.

Primary Cloudflare documentation and engineering article reviewed October 11, 2026.
