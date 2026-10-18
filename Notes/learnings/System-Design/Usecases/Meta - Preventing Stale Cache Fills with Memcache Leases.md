# Meta - Preventing Stale Cache Fills with Memcache Leases

## The Core Problem

A cache fill can finish after the data it fetched becomes obsolete. Facebook's [2013 McDipper engineering post](https://engineering.fb.com/2013/03/05/web/mcdipper-a-key-value-cache-for-flash-storage/) explains why writers invalidate cached entries rather than directly replace them: replacement writes can arrive out of order. However, deletion alone still permits an older, in-flight fill to repopulate the cache afterward.

Illustrative race: reader A fetches version 1; writer B stores version 2 and deletes the cached entry; A then caches version 1. The cache is populated, but wrong.

This note examines Facebook's historical **2013 Memcache lease design**, not a claim about Meta's current deployment.

## Architecture & Component Design

The [NSDI paper, §3.2.1](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf#page=4), describes two coordinated controls:

1. **Authorize the fill.** On a miss, Memcache can issue a key-bound 64-bit lease token. The client returns it with the fill. A delete invalidates the token, allowing the server to reject a delayed stale set.
2. **Throttle competing fills.** The documented default issues a token at most once per key per ten seconds. Other requests receive a notification to wait briefly and retry; successful fills usually finish within milliseconds.

This addresses both stale sets and thundering herds. For one measured group of herd-prone keys, leases reduced peak database queries from 17,000/s to 1,300/s—not a universal performance guarantee.

## Trade-offs & Bottlenecks

The paper also permits temporarily retained, explicitly stale values for applications that accept them. Waiting and freshness therefore remain application-level choices; the token mechanism is not a promise that every read returns the newest database value. [Paper, §3.2.1](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf#page=4).

Design implication: [[Idempotency]] alone is insufficient. Repeating “set version 1” can be idempotent while still overwriting newer state. Safe repetition and permission to publish are separate concerns.

## Key Takeaway

Design implication: treat cache repopulation as a coordinated write, not harmless cleanup after a miss. A useful review asks both “may this result still be published?” and “how many callers should rebuild it?”

Sources linked inline; reviewed October 18, 2026. The race and design implications are explanatory synthesis.
