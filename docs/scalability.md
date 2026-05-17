# Scalability

## 1. Current Scalability Model

The current engine uses a **single-process, cursor-driven, bounded-concurrency** model.

Scalability is achieved through:

- incremental page extraction (HubSpot cursor pagination)
- bounded parallel load workers (thread pool)
- low-memory batch execution

## 2. Bounded-Memory Behavior

Memory remains bounded because the runtime processes only one page window at a time.

- no full dataset materialization
- batch-sized working set only
- cursor progression externalizes position instead of retaining historical data

## 3. Concurrency Handling

Concurrent loading is implemented using fixed thread workers.

Benefits:

- improved throughput for network-bound calls
- controlled pressure on target API

Limitations:

- static worker count
- no adaptive throttle based on live API responses

## 4. Bottlenecks

Primary bottlenecks:

1. Monday API latency and complexity/rate limits
2. Per-record mutation overhead
3. Single-process execution boundary
4. Lack of retry/backoff for transient failure smoothing

## 5. API Limitations

Source and target platform constraints directly shape scalability:

- HubSpot page size and cursor traversal model
- Monday GraphQL mutation throughput and board complexity constraints
- external API timeouts and transient network faults

## 6. Throughput Constraints

Throughput is constrained by:

- request round-trip latency
- mutation cost per record
- worker count and target-side acceptance rate

Scaling beyond current ceiling requires architectural changes, not only parameter tuning.

## 7. Partitioning Opportunities

Partitioned extraction can unlock horizontal scaling:

- segment by source ID ranges
- segment by creation/update windows
- segment by object subsets or board routing

Each partition should have isolated checkpoint state to prevent overlap.

## 8. Distributed Execution Roadmap

To evolve from single-node runtime:

1. externalize checkpoint state (Redis/Postgres)
2. assign partitions to worker processes
3. introduce centralized coordinator or orchestrator
4. enforce idempotent write semantics

## 9. Async Roadmap

Async migration runtime can improve scalability by:

- reducing thread overhead
- enabling explicit semaphores/rate limiting
- supporting adaptive concurrency with response-aware backoff

Recommended target:

- async HTTP clients for both APIs
- unified concurrency governor
- central retry policy with jittered backoff

## 10. Centralized State Roadmap

Centralized state is required for robust multi-worker scaling:

- durable cursor/checkpoint registry
- run-level metadata and lease ownership
- recovery-safe worker failover

State backend candidates:

- Redis (fast coordination, ephemeral + persistence options)
- Postgres (durable transactional state, stronger auditability)

## 11. Scaling Strategy Summary

| Stage | Model | Scaling Characteristic |
|---|---|---|
| Current | single runner + local checkpoint | simple and reliable for moderate volume |
| Near-term | tuned worker/page sizing + retries | better stability under API variance |
| Mid-term | partitioned workers + centralized state | horizontal scale-out |
| Advanced | async + orchestrated + idempotent writes | platform-grade migration execution |
