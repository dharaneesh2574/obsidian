# Microsoft - Ordered Processing with Azure Service Bus Sessions

## The Core Problem

A workflow can require related messages to be handled in order while unrelated workflows run concurrently. Sending everything to one worker preserves serialization but sacrifices throughput. Letting arbitrary workers compete for every message can separate dependent steps across processors.

Azure Service Bus sessions provide an application-defined grouping and exclusive receiver ownership. This case covers the documented session and Peek-Lock contracts, not an assumption that ordered delivery makes external business effects exactly once.

## Architecture & Component Design

**Choose the ordering scope.** Producers set `SessionId` on related messages sent to a session-enabled queue or subscription. The application defines what the group represents and where its workflow ends. An illustrative choice is one session per order, rather than one session for the entire commerce application. Microsoft's [session guide](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-sessions) describes this as separating interleaved streams into ordered groups.

**Assign one receiver to each active group.** Accepting a session acquires an exclusive lock covering its messages, including later arrivals. Other sessions can have different owners, enabling [[Parallel Processing]] across groups. The receiver releases ownership by closing or losing the lock; renewal can extend ownership. Design implication: ordered delivery still requires the application to avoid launching dependent effects concurrently in ways that reorder their completion.

**Settle individual work explicitly.** In Peek-Lock mode, receiving is not deletion. After successful processing, `Complete` tells the broker to remove the message. `Abandon` releases it for another attempt; `DeadLetter` moves an unprocessable message aside. Microsoft's [settlement documentation](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-transfers-locks-settlement) distinguishes these outcomes from Receive-and-Delete, which accepts possible loss if the receiver fails after delivery.

An illustrative processing path is:

`accept session → receive → apply effect → complete message → receive next → close session`

**Retain recovery context.** Session state can hold an application-defined checkpoint or a reference to external state, available to the next owner. It persists until explicitly cleared and consumes entity storage. This helps a replacement receiver find progress; it does not define how to atomically coordinate a checkpoint with another database. [Session state](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-sessions).

**Design for uncertain completion.** Locks are volatile, and settlement can fail because of a lost connection or expired lock. Work may already have finished when completion fails. Microsoft recommends identifying repeated work with a message or business identifier and making effects idempotent. Its [duplicate-processing guide](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-message-loss-and-duplicates) also separates send-side duplicate detection from consumer [[Idempotency]]: filtering repeated sends does not prevent every receive-side redelivery.

## Trade-offs & Bottlenecks

- **Hot-group bottleneck — design implication:** adding receivers helps independent sessions, but does not remove serialization for one heavily used ordering key. Choose the narrowest grouping that preserves the business invariant.
- **Renewal versus failover:** a longer lock can accommodate slow work but delay recovery after a receiver stops. Renew legitimate long-running processing rather than treating ownership as permanent. [Lock management](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-transfers-locks-settlement).
- **Recovery can change order:** a dead-lettered message resubmitted to the original queue receives a new enqueue position; its earlier relative ordering is lost. Design implication: dependent workflows need an explicit repair policy, not blind reinsertion. [Session recovery limits](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-sessions).
- **Checkpointing is not a distributed transaction:** design implication: recording progress before the effect risks skipping unfinished work; recording it afterward risks repeating the effect. Use an atomic local update where possible and a safe retry protocol for external effects.

## Key Takeaway

Sessions make the ordering boundary explicit: serialize related work, scale across independent groups, and transfer ownership through locks. Reliable processing still needs deliberate settlement, recoverable progress, and duplicate-safe effects. Broker ordering and business correctness are related responsibilities, not interchangeable guarantees.

Sources linked inline; reviewed October 4, 2026. The order example and labeled design implications are explanatory synthesis.
