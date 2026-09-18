# Amazon - Preventing Lost Updates with S3 Conditional Writes

## The Core Problem

Two workers can read the same shared object, calculate different updates, and then overwrite each other. Checking the object before sending an unconditional write leaves a gap: another writer can change it between the check and the write. The last accepted replacement can silently erase the other worker's changes.

Amazon S3's [November 2024 conditional-write announcement](https://aws.amazon.com/about-aws/whats-new/2024/11/amazon-s3-functionality-conditional-writes/) moved the comparison into S3's write operation. Clients can submit the ETag they observed and require it to match before replacement. This note studies that public coordination contract for general purpose buckets, not S3's internal locking or replication implementation.

## Architecture & Component Design

**Separate creation from replacement.** `If-None-Match: *` asks S3 to create the object only if its key has no current object. `If-Match: <ETag>` asks it to replace an existing object only when its current ETag matches the supplied value. These are different preconditions, not interchangeable retry options. The [conditional-write guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html) documents both for `PutObject` and `CompleteMultipartUpload`.

**Compare at the write boundary.** An illustrative client sequence is:

`read object + ETag → compute replacement → conditional PUT → accept success or reconcile conflict`

Suppose workers A and B read ETag E0. A commits a replacement with a different ETag E1. B's later write conditioned on E0 fails instead of replacing A's result. As the [launch explanation](https://aws.amazon.com/about-aws/whats-new/2024/11/amazon-s3-functionality-conditional-writes/) describes, S3 performs the comparison before committing the write. The application still decides how to merge or abandon competing changes.

**Classify failures.** The [PutObject API](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObject.html) specifies `412 Precondition Failed` when the ETag condition is false. It also documents `409 ConditionalRequestConflict`; for an `If-Match` conflict, fetch the object's ETag again and retry the upload. Design implication: do not merely attach a fresh ETag to stale replacement bytes. Re-read the relevant content and recompute the intended update, or surface the conflict to the caller.

**Enforce participation.** Bucket policy can require conditional headers using `s3:if-match` or `s3:if-none-match`. Otherwise, a writer using an unconditional operation can ignore the application's coordination convention. AWS's [policy guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes-enforce.html) also requires accounting for multipart setup and part transfers: those APIs do not accept the headers, so examples use `s3:ObjectCreationOperation` to exempt them while constraining object creation. Policy enforcement is separate from conflict resolution.

## Trade-offs & Bottlenecks

- **Contention moves to clients.** Design implication: frequent changes to one shared object can force repeated reads and recomputation. Independent keys offer more scope for [[Parallel Processing]] than one heavily contested manifest.
- **Creation is not permanent uniqueness.** In a versioned bucket, `If-None-Match` considers the current version; a current delete marker permits creation. It is not a guarantee that the key has never existed. [Documented behavior](https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html).
- **Uploaded parts do not reserve a key.** Another writer can create the object before conditional multipart completion. This complements [[Amazon - Resumable Large Object Uploads with S3 Multipart Upload]]: successful byte transfer does not ensure final publication wins.
- **A lost response remains ambiguous.** Design implication: conditional writes do not replay a previously saved response like [[Stripe - Safe API Retries with Idempotency Keys]]. Applications still need [[Idempotency]] or reconciliation when they cannot tell whether their earlier write succeeded.

## Key Takeaway

Put the precondition where the mutation occurs. S3 can reject an obsolete replacement, but the client must handle conflicts and every relevant writer must follow the protocol. Conditional writes prevent a class of lost updates; they do not automatically merge changes or make an entire multi-object workflow transactional.

Sources linked inline; reviewed September 18, 2026. Client sequences and design implications are explanatory synthesis.
