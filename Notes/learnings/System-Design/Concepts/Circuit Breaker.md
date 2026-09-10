# Circuit Breaker

## What It Is

A circuit breaker temporarily stops calls to a dependency when recent outcomes indicate that continuing is likely to fail. It records execution results and rejects new attempts while open, giving the caller a fast failure or an opportunity to provide a fallback.

## Why It Exists

A slow dependency can hold threads and connections long enough to exhaust its callers. Repeatedly waiting for the same failure wastes capacity and can turn one service outage into a cascade. A breaker limits further exposure after detecting unhealthy behavior.

## How It Works

| State | Behavior | Next transition |
| --- | --- | --- |
| Closed | Permit calls and record selected outcomes | Open when the configured failure condition is met |
| Open | Reject protected calls without contacting the dependency | After a cooldown, permit a recovery trial |
| Half-open | Admit limited trial traffic | Close after sufficient success; reopen after failure |

Exact sampling windows and trial policies vary. Hystrix combines a minimum request volume with an error-percentage threshold; its documented recovery path admits one trial request after the sleep window. A failed trial starts another open interval.

## Trade-offs

- Aggressive thresholds can reject useful work during transient problems; conservative thresholds react slowly.
- Low traffic produces weak evidence, making minimum sample size important.
- A breaker does not replace timeouts or bounded concurrency: work can accumulate before the breaker opens.
- Stopping the caller's wait does not prove that remote execution stopped. Any subsequent retry must consider [[Idempotency]].
- Fallbacks need their own correctness and resource limits. Returning a default is inappropriate when it would falsely imply a successful write.

## Related

[[Connection Pooling]] retains transport resources; a circuit breaker decides whether another dependency call should be attempted. [[Idempotency]] governs the safety of repeating an operation.

Used in [[Netflix - Dependency Isolation with Hystrix]]. References: [Martin Fowler: Circuit Breaker, March 6, 2014](https://martinfowler.com/bliki/CircuitBreaker.html) · [Hystrix configuration](https://github.com/Netflix/Hystrix/wiki/Configuration) · [Hystrix execution and cancellation](https://github.com/Netflix/Hystrix/wiki/How-it-Works) · [Hystrix fallback semantics](https://github.com/Netflix/Hystrix/wiki/How-To-Use). Reviewed September 10, 2026.
