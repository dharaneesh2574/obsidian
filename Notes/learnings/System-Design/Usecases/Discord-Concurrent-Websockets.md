### The Core Problem — How Discord Handles Millions of Concurrent WebSocket Connections

Discord needs to deliver messages, typing indicators, presence changes, and voice-state updates to users almost instantly. Traditional HTTP polling would create enormous request overhead, so Discord maintains long-lived **WebSocket connections** between clients and its Gateway infrastructure.

The difficult part isn't simply accepting millions of sockets. Discord must efficiently determine **which connected users should receive each event**, while handling connection failures, reconnects, traffic spikes, and users connected from many devices.

### Architecture & Component Design

A simplified architecture looks like:

```
Discord Client
      │
   WebSocket
      │
      ▼
┌───────────────┐
│ Gateway Nodes │  ← millions of persistent connections
└───────┬───────┘
        │
        │ distributed event routing
        ▼
┌─────────────────────┐
│ Messaging / Services│
└─────────┬───────────┘
          │
          ▼
   Persistent Storage
```

Clients connect to **Gateway nodes**, which maintain WebSocket sessions and subscriptions.

Suppose Alice sends a message to a server containing Bob and Charlie. The message service persists the message and publishes an event. Discord's event-routing infrastructure determines which Gateway nodes currently hold connections for Bob and Charlie. Those gateways then push the event through the existing WebSockets.

This effectively separates:

**message processing → event routing → connection management**

That separation allows each layer to scale independently.

### Trade-offs & Bottlenecks

Persistent connections consume memory and file descriptors, so Gateway servers have a practical connection ceiling. Adding servers therefore requires **horizontal partitioning/sharding** of connections.

Another major issue is the **reconnect storm**. If a Gateway node containing hundreds of thousands of connections crashes, clients may simultaneously reconnect, potentially overwhelming healthy nodes. Systems therefore need randomized reconnect delays, session resumption, rate limiting, and careful load balancing.

Presence and typing indicators also illustrate an important consistency trade-off. Losing a `"user is typing"` event is usually acceptable, while losing an actual message is not.

So systems commonly treat them differently:

```
Message
→ persist reliably
→ then distribute

Typing / Presence
→ distribute quickly
→ occasional loss acceptable
```

This avoids paying strong durability and consistency costs for ephemeral information.

### Key Takeaway

**Not every real-time event deserves the same reliability guarantees.**

Large real-time systems become much easier to scale when durable business events such as messages are separated from ephemeral events such as presence and typing indicators. This allows the architecture to optimize independently for **durability, latency, and throughput**.