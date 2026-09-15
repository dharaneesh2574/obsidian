# Amazon - Resumable Large Object Uploads with S3 Multipart Upload

## The Core Problem

Sending a large object as one request makes a network interruption expensive: work may need to restart from the beginning. A single transfer can also underuse available bandwidth. Amazon S3's multipart upload API separates transferring bytes from creating the completed object, allowing independently uploaded parts to survive an interrupted transfer.

This case studies the public protocol for general purpose buckets, not undocumented S3 storage internals. It addresses upload recovery and completion rather than the transcoding pipeline in [[YouTube - Video Processing Pipeline]].

## Architecture & Component Design

**Create a named upload session.** Initiation returns an upload ID. Subsequent operations identify that upload, while part numbers identify each piece and its position in the resulting object. Uploading parts is not equivalent to creating separately readable application objects. The [multipart overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html) describes initiation, part upload, and completion as separate stages.

**Transfer and retry bounded pieces.** Parts can arrive independently and out of order, enabling [[Parallel Processing]] across transfers. A failed part can be retried without retransmitting successful parts. The [UploadPart contract](https://docs.aws.amazon.com/AmazonS3/latest/API/API_UploadPart.html) states that uploading the same part number again replaces its prior contents within that upload. This offers a useful scoped connection to [[Idempotency]]: repeating identical bytes into the same slot does not append an extra piece. It is not a promise that every multipart operation deduplicates arbitrary retries.

**Retain a completion manifest.** The client keeps part numbers and returned ETags, then submits the intended list with the upload ID. S3 assembles the listed parts in ascending part-number order. The [completion API](https://docs.aws.amazon.com/AmazonS3/latest/API/API_CompleteMultipartUpload.html) requires the client to supply a complete list; S3 does not know the application's intended file merely from the parts it has received.

The source-derived lifecycle is:

`initiate → upload/retry parts independently → submit ordered manifest → confirm completion`

**Validate the result, not just the status line.** Completion may take several minutes. S3 can send initial `200 OK` headers and later embed an error in the response body. A direct API client must parse the response; AWS SDKs handle that embedded-error condition. Declaring success at the first HTTP status would confuse accepted processing with a completed object.

**Check data integrity explicitly.** The final ETag is not necessarily the object's MD5 digest. S3 supports checksum validation, including full-object and composite checksum modes with algorithm-specific rules. Its [integrity guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/checking-object-integrity-upload.html) explains that S3 independently calculates and checks supplied checksum values. A checksum mismatch is an integrity failure, not evidence that another blind completion attempt will fix incorrect bytes.

## Trade-offs & Bottlenecks

**Design implications:** smaller parts reduce retransmission cost but increase request and manifest overhead. Excessive concurrency can move the bottleneck to client memory, local disk, or network capacity. A resumable client should durably retain its upload ID, source identity, and acknowledged part metadata; retrying from a changed local file risks mixing versions.

Incomplete uploads also consume billable storage. Completion or abort releases the stored-part allocation; lifecycle rules can clean up abandoned uploads. The [abort API](https://docs.aws.amazon.com/AmazonS3/latest/API/API_AbortMultipartUpload.html) warns that already in-flight part uploads may still succeed. Cleanup can require repeated aborts and verification with `ListParts`, so cancelling local workers alone does not prove remote cleanup finished.

## Key Takeaway

Make the retry unit smaller than the final artifact, but keep finalization explicit. S3's protocol combines independent transfers, stable part identity, a completion manifest, integrity checks, and cleanup. Reliable clients must account for all five, not merely launch uploads concurrently.

Sources linked inline; reviewed September 15, 2026.
