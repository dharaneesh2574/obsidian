# Parallel Processing

## What It Is

Parallel processing divides a large computation into independent units that can run at the same time on multiple CPU cores, machines, or accelerators.

## Why It Exists

Some jobs are too slow or too large for one worker. If their parts can be processed independently, adding workers can reduce completion time and increase system throughput.

## How It Works

A coordinator partitions an input into tasks, distributes them to workers, tracks completion, and combines the outputs. For example, a video can be split at safe frame boundaries, each segment encoded independently, and the encoded segments reassembled in order.

The speedup is limited by work that cannot be parallelized, coordination overhead, uneven task sizes, and the slowest worker. Systems often use retries and smaller tasks to reduce the impact of failed or slow workers.

## Trade-offs

Parallelism lowers latency and scales throughput, but adds scheduling, synchronization, retry, and result-assembly complexity. More workers also increase resource cost and may overload shared storage or networks. Tasks must be deterministic or safely repeatable when retries are possible.

## Related Concepts

[[Hardware Acceleration]]
