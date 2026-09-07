# Idempotency

## What It Is

Idempotency means repeating an operation has the same intended effect as applying it once. Setting a value to 10 is idempotent; incrementing it by 10 is not. Identical response bytes are not required by the general property, although an API can offer response replay as an additional guarantee.

## Why It Exists

A timeout does not tell a caller whether the server completed its work. Retrying could duplicate a side effect; refusing to retry could leave the operation unfinished. Idempotency makes retries safe within a defined scope, so temporary communication failures need less manual reconciliation.

## How It Works

An operation can be naturally idempotent, or the client can assign a unique identity to one logical request and reuse it on retries. Two intentional operations need different identities even when their payloads match.

A service implementing request deduplication records that identity, the request parameters, and its progress or result. Repeated requests either obtain the previous result or follow the documented policy for work still in progress. Changing the parameters under the same identity should be rejected.

Recording the identity and applying the effect must be coordinated. Within one database, an atomic transaction can couple those changes. A separate external side effect needs its own safe retry or recovery protocol; writing a token before or after that effect alone leaves a crash window.

## Trade-offs

- Remembering request identities consumes storage; expiration limits how long a delayed retry can be recognized.
- Concurrent duplicates require coordination, potentially causing contention or retryable conflicts.
- An idempotency guarantee prevents repeated effects, but does not guarantee success or eliminate network requests.
- Each layer must preserve logical identity. Replaying an event from a [[Partitioned Log]] with a fresh idempotency key can still repeat the effect.

## Related

- [[Partitioned Log]] — consumer recovery can replay work after a checkpoint gap.
- [[Parallel Processing]] — duplicate attempts can overlap and require atomic coordination.
- [[Stripe - Safe API Retries with Idempotency Keys]] — response replay and bounded deduplication for mutating API requests.
- [Amazon Builders' Library: Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)
- [Stripe: Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency)
