# GitHub - Webhook Ingestion and Delivery Recovery

## The Core Problem

A repository event may trigger a slow build, synchronize another service, or update application state. Running that work inside the webhook HTTP request couples delivery success to downstream latency. A timeout also leaves an ambiguous outcome: the receiver might have started work even though GitHub records a failed delivery.

GitHub's documented webhook contract separates fast receipt from application processing. This case covers that contract and a receiver design derived from it; it does not claim that GitHub internally uses the queue or database described below.

## Architecture & Component Design

**Authenticate the original body.** Configure a secret, then verify `X-Hub-Signature-256` using HMAC-SHA256 over the received payload. Compare signatures with a constant-time function, and do not parse and reserialize the body before verification. GitHub's [validation guide](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries) supplies examples using the original request body and warns against proxies modifying the payload. Keep the secret outside source control. Signature verification establishes payload authenticity and integrity; it is separate from deciding whether processing already happened.

**Keep receipt short.** GitHub expects a `2XX` response within ten seconds and recommends a queue for asynchronous processing. Route only supported event/action combinations using `X-GitHub-Event` and the payload's `action` field. These are documented [webhook best practices](https://docs.github.com/en/webhooks/using-webhooks/best-practices-for-using-webhooks), not a requirement to finish a build within the response window.

**Design implication: make acceptance durable.** A robust receiver can save the verified payload and delivery identity in a durable inbox before returning success. Workers then claim pending entries and perform the application work. Responding successfully before retaining the work creates a crash window where GitHub sees success but the event disappears locally. Persisting an inbox entry is a proposed receiver implementation, not a documented GitHub storage mechanism.

An illustrative path is:

`GitHub → verify payload → persist inbox entry → return 2XX → worker processes entry`

**Preserve delivery identity.** Requested redelivery retains the original `X-GitHub-Delivery` value. Design implication: use that identity to recognize repeated delivery attempts, while recording whether work is pending, completed, or failed. [[Idempotency]] still requires coordinating the business effect with its completion record. Merely inserting a “seen” marker and skipping all later attempts can lose unfinished work after a crash. Independent inbox entries can use [[Parallel Processing]], subject to application ordering needs.

**Build the recovery path explicitly.** GitHub [does not automatically redeliver failures](https://docs.github.com/en/webhooks/using-webhooks/handling-failed-webhook-deliveries). It documents a scheduled recovery script that lists attempted deliveries since its previous run, identifies failures, and requests redelivery through the REST API. Manual redelivery is also supported. The documented delivery-inspection APIs cover repository, organization, and GitHub App webhooks, not Marketplace or Sponsors webhooks.

## Trade-offs & Bottlenecks

- **Two success boundaries:** HTTP acceptance and completed business work are different states. Design implication: monitor pending inbox age and worker failures as well as endpoint responses; a healthy HTTP endpoint can conceal a stalled worker pool.
- **Two recovery paths:** delivery reconciliation repairs failures visible to GitHub. Once the receiver has returned success, subsequent worker failures require local retry or investigation. The remote delivery status does not track those application outcomes.
- **Storage and coordination cost:** design implication: a durable inbox adds writes, retention decisions, and duplicate-handling logic. An external side effect still needs its own safe retry protocol; a local database transaction cannot automatically include it.
- **Repeated failure needs diagnosis:** GitHub recommends investigating recurrent delivery failures rather than only resending them. Recovery code is another operational component, not a substitute for fixing authentication, reachability, or timeout problems.

## Key Takeaway

Treat receipt, processing, and recovery as separate responsibilities. GitHub's signature and delivery identity help establish what arrived; fast acknowledgment avoids coupling delivery to slow work. Durable local acceptance and deliberate reconciliation complete the receiver design without pretending that a successful HTTP response means the business operation finished.

Sources linked inline; reviewed September 20, 2026. The inbox architecture and labeled design implications are explanatory synthesis.
