# Reliability

## 1. Reliability Objectives

This migration engine is designed to optimize:

- deterministic progression
- recoverable long-running execution
- schema integrity during transfer
- explicit failure visibility

## 2. Fault Tolerance Model

Fault tolerance is implemented through restart-safe progression rather than in-process self-healing.

Core mechanisms:

- checkpointed cursor state
- bounded batch execution window
- explicit exception propagation

## 3. Recovery Guarantees

Current guarantees:

- migration resumes from last committed checkpoint after interruption
- forward progress is preserved at batch boundary
- full restart replay is avoided in normal recovery

Non-guarantees:

- record-level exactly-once delivery is not guaranteed without idempotent target semantics

## 4. Operational Reliability

Reliability controls in code path:

- required token presence checks
- strict mapping key validation
- HTTP status checks (`raise_for_status`)
- GraphQL error payload checks

These controls intentionally stop execution on integrity threats.

## 5. Interruption Recovery

Interruption handling relies on re-execution:

- process crash -> rerun process
- host restart -> rerun process
- transient network issue -> rerun process

Checkpoint state anchors recovery point across runs.

## 6. Migration Durability

Durability is tied to checkpoint commit timing:

- commit after successful batch completion
- no commit on partial-batch failure

This preserves deterministic replay boundaries and protects against skipped data.

## 7. Failure Handling Behavior

| Failure Type | Current Handling | Reliability Outcome |
|---|---|---|
| Missing credentials | immediate runtime failure | prevents undefined API behavior |
| Mapping mismatch | immediate failure | prevents silent schema drift |
| HubSpot request failure | exception propagation | preserves prior checkpoint |
| Monday mutation error | exception propagation | blocks incorrect progression |

## 8. Integrity Protections

- no silent mapping fallback for unknown fields
- no ignored GraphQL error payloads
- constrained progression through explicit checkpoints

Integrity is prioritized over completing a run with partial unknown correctness.

## 9. Retry Strategy Gaps

Current implementation does not provide:

- automatic retry with backoff
- transient error classification
- dead-letter handling for poisoned records

Impact:

- manual reruns required for transient issues
- lower resilience under unstable network/API conditions

## 10. Deterministic Progression Behavior

Progression behavior is deterministic at page boundary:

1. fetch page
2. process page
3. commit cursor

This sequencing ensures migration state reflects completed work, not intended work.

## 11. Reliability Evolution

Recommended enhancements:

1. transient retry with exponential backoff + jitter
2. DLQ for per-record isolation failures
3. structured event logging with correlation IDs
4. centralized checkpoint transaction semantics
5. idempotent upsert semantics for strict replay safety
