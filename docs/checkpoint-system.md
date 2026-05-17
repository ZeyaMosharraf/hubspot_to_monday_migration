# Checkpoint System

## 1. Purpose

The checkpoint system provides **progress durability** for long-running migration execution.

Without checkpointing, any interruption requires full replay from the beginning. With checkpointing, restart resumes from the last committed source cursor boundary.

## 2. Checkpoint Semantics

- Checkpoint stores a single value: `after` cursor from HubSpot pagination.
- Checkpoint represents the latest **successfully completed batch boundary**.
- State format is JSON in `state/checkpoint.json`.

Example:

```json
{
  "after": "123456789"
}
```

## 3. Cursor Progression Model

1. Read existing cursor at process start.
2. Fetch batch using cursor as `after`.
3. Process and load current batch.
4. Persist returned `next_after` as new checkpoint.
5. Repeat.

This creates monotonic progression through source pages.

## 4. Restart Behavior

On restart:

- runtime loads saved cursor
- extraction resumes at saved boundary
- migration continues forward without full replay

If checkpoint is `null`, migration starts from initial page.

## 5. Recovery Lifecycle

| Phase | Checkpoint Behavior | Operational Result |
|---|---|---|
| Startup | Read last saved cursor | establish resume point |
| Batch execution | hold in-memory `next_after` | no commit yet |
| Batch success | write checkpoint file | progress committed |
| Failure before write | keep previous cursor | replay of last uncommitted batch |

## 6. Replay Prevention Behavior

Checkpointing minimizes replay scope to the most recent uncommitted batch window.

It does not guarantee record-level dedupe by itself; strict replay prevention at record granularity requires idempotent target write semantics.

## 7. Interruption Handling

Interruption classes:

- process termination
- machine restart
- network/API outage
- unexpected runtime exception

In all cases, persisted checkpoint enables continuation from last commit boundary rather than from zero.

## 8. Checkpoint Write Timing

Checkpoint is written **after batch processing completes**.

Why timing matters:

- writing too early can skip unprocessed data
- writing too late increases replay scope
- current timing ensures batch-level durability with deterministic forward progression

## 9. Failure Scenarios

| Scenario | Outcome | Recovery |
|---|---|---|
| HubSpot fetch fails | no new batch processed | restart retries same cursor |
| Monday load fails mid-batch | checkpoint not advanced | replay batch on restart |
| Checkpoint file missing | treated as first run | start from beginning |
| Corrupted checkpoint JSON | startup failure | repair file manually, then rerun |

## 10. Operational Value

Checkpointing solves the core migration reliability problem: **long-run progress loss under interruption**.

Business impact:

- reduces restart overhead
- limits repeated API calls
- improves predictability of migration windows
- supports safer execution for large datasets

## 11. Design Constraints

- single-file checkpoint model is optimized for single-runner execution
- not designed for multi-worker distributed coordination
- does not currently encode batch metadata, run IDs, or audit trail

## 12. Evolution Path

Recommended progression:

1. move checkpoint state to Redis/Postgres
2. include run metadata (batch ID, timestamp, status)
3. add checkpoint integrity validation
4. support partition-level checkpoint state for distributed workers
