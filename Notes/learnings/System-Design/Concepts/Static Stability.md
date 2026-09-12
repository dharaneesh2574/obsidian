# Static Stability

## What It Is

Static stability is the ability to keep operating through a specified dependency failure using resources and configuration already in place. It does not mean that no runtime decisions occur; it means recovery does not require the impaired dependency to make new resources or configuration available.

## Why It Exists

An emergency recovery plan can accidentally depend on the system that is failing. Launching replacement servers, fetching configuration, or changing DNS records introduces management dependencies precisely when they may be least reliable.

## How It Works

Separate the control plane, which creates and changes resources, from the data plane, which performs their ongoing work. Preserve enough state in the data plane to continue operating when control-plane updates stop. Provision recovery capacity and routing choices before an incident.

For example, a multi-zone service can reserve enough capacity in its surviving zones to carry the workload without launching new instances after losing a zone. Health-based routing can still change which existing servers receive work.

## Trade-offs

- Spare capacity costs money and must be tested under realistic load.
- Existing configuration can remain usable while updates are unavailable; this does not guarantee configuration freshness.
- The guarantee is failure-specific: independence from a management API does not protect against losing all serving capacity.

## Related

[[Circuit Breaker]] limits calls to an unhealthy dependency; static stability removes the need for selected dependencies during continued operation or recovery. [[Shuffle Sharding]] limits failure exposure, a different and complementary concern.

Used in [[Amazon - Route 53 DNS Failover Without Control-Plane Changes]]. References: [AWS static stability](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/static-stability.html) · [Amazon Builders' Library: Static stability using Availability Zones](https://aws.amazon.com/builders-library/static-stability-using-availability-zones/). Reviewed September 12, 2026.
