# Stripe - Safe API Retries with Idempotency Keys

## The Core Problem

A client sends a payment request, Stripe processes it, and the response disappears during a network failure. The client sees a timeout without knowing whether the operation happened. Blindly submitting another payment could charge twice; abandoning the attempt leaves its outcome unresolved.

Stripe's API combines client-generated request identities with server-side [[Idempotency]] handling. This note covers the documented API v1 contract, checked in September 2026, and the design rationale in Stripe's engineering article. It does not assume a particular internal database or replication topology.

## Architecture & Component Design

The client supplies an `Idempotency-Key` with a mutating `POST` request. That key identifies one logical operation across multiple transmission attempts. For a network retry, the client sends the same key and parameters. Stripe compares reused keys against the original parameters and rejects mismatches.

Once endpoint execution begins, Stripe saves the resulting HTTP status and response body. A completed duplicate request receives that saved result, including a saved `500` error. Validation failures and conflicts with a concurrently executing request do not create a saved result when execution has not begun; those requests can be retried.

An illustrative lost-response sequence is:

```text
Client → Stripe: request with key K
Stripe: execute operation and retain its response
Stripe → Client: response lost in transit
Client → Stripe: same request, same key K
Stripe → Client: return the retained response
```

The retry schedule matters too. Stripe's engineering article pairs idempotency with exponential backoff and random jitter: repeated failures increase the wait, and randomness spreads clients' next attempts. This controls the extra load retries place on an already struggling service.

A server error needs a different interpretation from a lost connection. Stripe documents `500` outcomes as indeterminate because side effects may already exist. A fresh key can initiate another operation, so changing keys simply to escape a cached error is unsafe. Stripe may reconcile the failed operation later and emit webhooks for objects created through that process; the original cached response remains unchanged.

The application therefore needs both a retry path and a way to reconcile its own state with the operation's eventual outcome. Repeatedly receiving an error is not evidence that no business effect occurred.

## Trade-offs & Bottlenecks

- **Bounded memory:** Stripe allows keys to be pruned after they are at least 24 hours old. Reusing a key after pruning creates a new request. The retention policy bounds deduplication rather than providing permanent protection.
- **Identity across layers:** A worker replaying a [[Partitioned Log]] event must preserve the logical operation's identity when calling an external API. A new key on every delivery defeats deduplication. This is a general integration implication, not a claim about Stripe's event transport.
- **Concurrent attempts:** [[Parallel Processing]] can produce overlapping duplicates. The API's conflict behavior avoids treating unfinished work as a completed cached response; callers must handle that response appropriately.
- **Coordination cost — design implication:** The server must keep request identity and business effects consistent through failures. A token lookup alone cannot close the gap between executing an effect and remembering it. The cited sources do not establish Stripe's exact storage mechanism.

## Key Takeaway

Safe retries depend on preserving the identity of an operation and understanding what the server remembers about it. Stripe combines that identity with response replay, parameter checks, and controlled retry timing. Its handling of retained errors and expired keys shows why idempotency is a precise, bounded API contract rather than a blanket promise that every workflow succeeds exactly once.

Sources: [Stripe engineering: Idempotency](https://stripe.com/blog/idempotency) · [Stripe API: Idempotent requests](https://docs.stripe.com/api/idempotent_requests) · [Stripe: Advanced error handling](https://docs.stripe.com/error-low-level) · [Amazon Builders' Library: Atomicity and request identity](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/). The AWS article supports the general design implication, not a description of Stripe's implementation.
