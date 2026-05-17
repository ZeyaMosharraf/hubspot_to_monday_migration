# Performance

## 1. Performance Model

The runtime is optimized for **network-bound API migration**, not CPU-intensive transformation.

Performance strategy:

- process in source pages
- parallelize target API writes with bounded workers
- persist progress incrementally to reduce restart overhead

## 2. Bounded Worker Pool Strategy

A fixed `ThreadPoolExecutor` worker pool is used for concurrent loads.

Benefits:

- improved request overlap
- controlled API pressure
- predictable resource usage

Tuning axis:

- `max_workers` relative to target API constraints and observed latency

## 3. Throughput Optimization

Current throughput optimization levers:

- page size (`HUBSPOT_PAGE_LIMIT`)
- worker count in thread pool
- minimizing per-record transform overhead

Throughput is typically constrained by external API behavior, not local compute.

## 4. Network-Bound Concurrency

Because each record load requires external round trips:

- concurrency directly improves wall-clock performance up to API limit boundary
- excessive concurrency can increase failures due to rate/complexity pressure

Balanced concurrency is required for stable sustained throughput.

## 5. Memory Management

Memory is constrained by:

- one page of source data in memory
- active in-flight tasks for current batch

No unbounded accumulation of processed records occurs.

## 6. Incremental Progression Impact

Incremental checkpoint progression improves effective performance in operational runs by:

- reducing recovery replay work after interruption
- avoiding full reruns for long migrations

This does not increase instantaneous throughput, but improves end-to-end migration completion efficiency.

## 7. API-Pressure Reduction

Current pressure controls:

- bounded worker count
- page-limited extraction
- immediate stop on error (prevents uncontrolled failure cascades)

Missing pressure controls:

- dynamic throttling
- adaptive backoff on transient API failures

## 8. Scaling Bottlenecks

Primary bottlenecks:

1. Monday per-request mutation overhead
2. target API rate/complexity ceilings
3. single-process execution architecture
4. absence of async backpressure and retry policy

## 9. Future Async Optimization

Async evolution opportunities:

- switch to async HTTP clients
- shared concurrency semaphore
- adaptive concurrency controller
- jittered exponential backoff

Expected gains:

- better latency hiding
- lower thread overhead
- more stable high-throughput operation under API variability

## 10. Performance Observability Targets

Recommended metrics:

- records processed per second
- request latency percentiles per API
- error rates by stage (extract/transform/load)
- retries and backoff utilization
- batch completion duration

These metrics convert tuning from guesswork to controlled optimization.
