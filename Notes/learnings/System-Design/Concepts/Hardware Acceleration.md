# Hardware Acceleration

## What It Is

Hardware acceleration moves a compute-heavy operation from general-purpose CPUs to specialized hardware designed for that workload, such as GPUs, TPUs, or video-encoding ASICs.

## Why It Exists

General-purpose processors are flexible, but repeated operations such as matrix multiplication, encryption, or video encoding can consume too much time, power, and data-center capacity at scale.

## How It Works

Software sends supported jobs to an accelerator whose data paths and execution units are optimized for those operations. A scheduler must still feed the device, move data, handle failures, and fall back when a requested operation is unsupported. The whole system must be balanced; accelerating computation alone can expose storage, memory, or network bottlenecks.

## Trade-offs

Specialized hardware can deliver much higher performance per watt and lower unit cost, but requires large upfront investment, specialized tooling, and long hardware-development cycles. It is less flexible than CPUs, can create vendor or architecture lock-in, and may become obsolete when algorithms or standards change.

## Related Concepts

[[Parallel Processing]]
