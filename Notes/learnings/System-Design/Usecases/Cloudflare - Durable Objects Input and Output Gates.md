# Cloudflare - Durable Objects Input and Output Gates

## The Core Problem

Single-threaded JavaScript does not automatically make an entire request atomic. When a handler awaits asynchronous work, another request can run before it resumes. Two counter updates can therefore read the same old value and overwrite each other. A second problem is durability: updating memory and constructing a success response is unsafe if the underlying write can still fail.

Cloudflare addresses these different boundaries with input and output gates. This case follows its [2021 runtime design](https://blog.cloudflare.com/durable-objects-easy-fast-correct-choose-three/), its later SQLite implementation, and current documentation—not a claim that every asynchronous function becomes a transaction.

## Architecture & Component Design

**Choose a coordination unit.** Each Durable Object is a globally unique, single-threaded instance with private persistent storage. Requests that need to coordinate around one room, document, or other entity go to that object's identity. Independent entities can use different objects. Cloudflare's [design guide](https://developers.cloudflare.com/durable-objects/best-practices/rules-of-durable-objects/) recommends modeling objects around the smallest useful unit of coordination, rather than sending the entire application through one global object.

**Input gates control event delivery.** During an asynchronous storage operation, the runtime defers other incoming events while allowing storage completions. In the storage-only counter example, a handler reads the counter, increments it, and writes it back without another request slipping between those steps. The gate does not serialize every operation the application explicitly starts: two counter-update functions launched concurrently by the same event can still race. An arbitrary external-network await is also not a protected storage operation. These boundaries are explicit in the [original explanation](https://blog.cloudflare.com/durable-objects-easy-fast-correct-choose-three/).

**Output gates control external visibility.** The application can continue executing after initiating a write, but outgoing network messages are held until pending writes are confirmed. Cloudflare's [SQLite engineering account](https://blog.cloudflare.com/sqlite-in-durable-objects/) explains that a response can be constructed before durability confirmation without being delivered early. If the write fails, the success response does not escape; the runtime returns an error and restarts the object. Its illustrated sequence separates application progress from the later release of the response.

**Synchronous SQL removes some yield points.** SQLite runs as a library in the object's own thread, and its query API returns synchronously. That avoids a network round trip per query and prevents other JavaScript from interleaving during uninterrupted SQL-and-application execution. It does not remove the need to confirm durability before releasing output. [SQLite implementation](https://blog.cloudflare.com/sqlite-in-durable-objects/).

**Initialization has an explicit barrier.** `blockConcurrencyWhile` can keep requests out until constructor initialization completes. Cloudflare's [state API documentation](https://developers.cloudflare.com/durable-objects/api/state/) says ordinary storage handling rarely needs this wrapper: storage gates or synchronous SQL already cover it. Keep the callback short; an exception or exceeding its 30-second timeout resets the object.

## Trade-offs & Bottlenecks

- **One hot object remains bounded.** One object's execution uses one thread. [[Parallel Processing]] comes from distributing independent coordination units across objects, not automatically splitting one object's shared state. [Scaling discussion](https://blog.cloudflare.com/sqlite-in-durable-objects/).
- **Durable acknowledgment still takes time.** A fast local query does not imply an equally fast externally visible response; output may wait for storage confirmation.
- **Design implication: retries still need identity.** If a durable update succeeds but its response is lost in transit, a retry can repeat the effect. [[Idempotency]] remains an application concern; gates are not an end-to-end exactly-once protocol.
- **Design implication: scope matters.** Local event admission and durable output ordering do not, by themselves, atomically coordinate another object or an external service.

## Key Takeaway

Separate three questions: what may interleave, when application code may continue, and when success may become externally visible. Cloudflare makes common local storage patterns safer by enforcing those boundaries in the runtime, while leaving application partitioning and distributed retry semantics explicit.

Sources linked inline; reviewed September 17, 2026.
