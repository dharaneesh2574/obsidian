# Google - Preventing Stale Authorization with Zanzibar

## The Core Problem

An editor removes a contractor from a budget document, then adds the confidential bids. A stale permission check could let the contractor read those bids. Google calls this the *new enemy* problem: an earlier permission is paired with later content. This example illustrates the failure described in the [2019 Zanzibar paper, §2.2](https://www.usenix.org/system/files/atc19-pang.pdf#page=4).

## Architecture & Component Design

Zanzibar uses Spanner's [[External Consistency]] to order permission writes. Before saving an edit, the application asks Zanzibar to authorize the change at the latest snapshot. If authorized, Zanzibar returns an opaque token, a *zookie*. The application commits that token and the new content together in its own store.

When serving that version, the application supplies its zookie with the viewer's authorization check. Zanzibar chooses a consistent permissions snapshot **at least as recent as** the token. In the example, the snapshot cannot predate the contractor's removal. The token sets a lower bound; it neither grants access nor freezes permissions at the instant of editing. [Check protocol, §2.4.4](https://www.usenix.org/system/files/atc19-pang.pdf#page=6)

## Trade-offs & Bottlenecks

Demanding the latest permissions for every read would add synchronization latency. A lower bound lets Zanzibar serve checks from replicated data when sufficiently fresh. The client must preserve the token-to-version pairing.

## Key Takeaway

Design implication: test the pairing, not just the policy. A permissions test against current data can pass while a content-serving path still supplies yesterday's token with today's document.
