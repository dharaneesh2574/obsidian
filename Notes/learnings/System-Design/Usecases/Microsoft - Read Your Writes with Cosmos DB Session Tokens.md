# Microsoft - Read Your Writes with Cosmos DB Session Tokens

## The Core Problem

Illustrative scenario: a user saves a profile through web server A, then refreshes through server B. The write succeeded, but B's database client might read a replica that has not caught up. Load balancing has separated the user's workflow from the client instance that remembers the write.

Azure Cosmos DB's session consistency addresses this through session tokens. This note covers Microsoft's documented NoSQL client contract, not an assumed internal replication algorithm.

## Architecture & Component Design

**Carry a freshness requirement.** Microsoft's [consistency-level documentation](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels#session-consistency) describes a token returned after a write and sent with subsequent reads. A replica must have data meeting that token or newer; otherwise the client retries another replica, and potentially another available region. The token is a minimum version barrier, not a request for an exact historical version.

**Let the SDK maintain local session state.** The [session-management guide](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-manage-consistency#utilize-session-tokens) explains that `CosmosClient` tracks tokens automatically. Separate web nodes with separate client instances do not automatically inherit one another's session state merely because they serve the same user.

**Transfer context across nodes.** After A completes the write, capture the response token and carry it through the application workflow to B's subsequent read request. Microsoft illustrates returning it through a cookie or another mechanism. In .NET, the documented example reads `response.Headers.Session` and sets `ItemRequestOptions.SessionToken` on the read. Tokens are partition-bound: use the latest token relevant to the items being read, not one universal token for the whole database.

**Keep the boundary explicit.** Session read-your-writes assumes one writer session or token sharing among writers. Passing tokens between client instances must preserve their contents. By contrast, [[External Consistency]] concerns a globally ordered transaction history; this session mechanism does not establish global serializability or a cross-partition snapshot.

## Trade-offs & Bottlenecks

A recreated client loses its cached tokens. Without relevant session context, reads can behave like eventual consistency. Design implication: test restart and load-balancer paths, not only write-then-read calls using one long-lived client. Replica retries can also add work when the initial destination cannot satisfy the token.

Do not interpret every missing-item response as replication lag. Microsoft's [not-found troubleshooting guide](https://learn.microsoft.com/en-us/azure/cosmos-db/troubleshoot-not-found) separates cross-client races and unavailable read sessions from incorrect partition-key/ID combinations, TTL expiry, and deleted resources. Token propagation addresses session continuity; it cannot repair an invalid lookup or restore deleted data. The guide recommends checking SDK versions for read-session-unavailable errors.

## Key Takeaway

Design implication: a user's logical session can span several stateless web servers, but its database consistency context still has to travel. Make that handoff explicit wherever later reads depend on earlier writes.

Sources linked inline; reviewed November 1, 2026.
