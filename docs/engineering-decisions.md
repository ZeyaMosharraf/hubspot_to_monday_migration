# Engineering Decisions (ADR)

## ADR-001: Threaded Concurrency vs Async Runtime

### Context
Load path is network-bound and dominated by API latency.

### Problem
Increase throughput without overcomplicating execution model.

### Decision
Use bounded `ThreadPoolExecutor` for concurrent record loading.

### Why This Approach
- simple operational footprint
- native fit for I/O-bound requests
- adequate for current migration scale

### Tradeoffs
- less fine-grained backpressure than async rate-limited runtime
- thread context overhead
- harder to coordinate advanced flow control

### Future Improvements
- adopt async client + semaphore/rate limiting
- implement adaptive concurrency based on error/latency signals

---

## ADR-002: Fail-Fast Integrity vs Skip-and-Continue

### Context
Migration correctness is more critical than maximum completion percentage.

### Problem
Decide error policy for mapping/API failures.

### Decision
Fail immediately on mapping misses and API errors.

### Why This Approach
- prevents silent data corruption
- preserves explicit operational signal on integrity issues
- simplifies post-run trust in migrated dataset

### Tradeoffs
- lower tolerance for malformed edge records
- may require manual intervention to continue

### Future Improvements
- add configurable policy modes (strict / tolerant)
- route malformed records to DLQ for isolated replay

---

## ADR-003: File Checkpoint State vs Distributed State Store

### Context
Need resumability for long-running migration in current single-runner model.

### Problem
Persist progress with minimal operational overhead.

### Decision
Use JSON file checkpoint (`state/checkpoint.json`) storing source cursor.

### Why This Approach
- minimal dependencies
- straightforward recovery behavior
- easy local operation and debugging

### Tradeoffs
- not multi-worker safe
- no built-in locking/versioning
- limited auditability

### Future Improvements
- migrate state to Redis/Postgres
- add transactional checkpoint writes
- include run metadata and status history

---

## ADR-004: Bounded Concurrency

### Context
Target API has practical throughput and complexity limits.

### Problem
Avoid API saturation while improving execution speed.

### Decision
Run loader with fixed worker bound (`max_workers=6`).

### Why This Approach
- protects against unbounded request fan-out
- enables controlled throughput tuning
- reduces operational instability risk

### Tradeoffs
- fixed limit may underutilize capacity in good conditions
- may still exceed constraints in adverse conditions without dynamic throttling

### Future Improvements
- dynamic concurrency controller
- quota-aware limiter driven by response telemetry

---

## ADR-005: Explicit Mapping Abstraction

### Context
Source property names and target column IDs evolve independently.

### Problem
Prevent mapping logic from being scattered across loader code.

### Decision
Centralize mapping definitions in `config/hubspot_columns.py` and `config/monday_columns.py`.

### Why This Approach
- clear schema contract surface
- simplifies change management
- improves maintainability and reviewability

### Tradeoffs
- manual mapping maintenance overhead
- incorrect mapping config can block runs

### Future Improvements
- add preflight mapping validation against live Monday board schema
- generate mapping diff reports

---

## ADR-006: Transform Layer Separation

### Context
Source/target schema conversion includes normalization concerns.

### Problem
Avoid API client coupling with business-level data shaping.

### Decision
Use dedicated transformer module (`transformers/company_mapper.py`).

### Why This Approach
- isolates normalization logic
- supports extension for new object types
- keeps clients focused on API transport concerns

### Tradeoffs
- extra module boundary to maintain
- requires discipline to keep concerns separated

### Future Improvements
- typed transform contracts (Pydantic/dataclasses)
- schema validation before load

---

## ADR-007: Incremental Processing Model

### Context
Dataset size can exceed comfortable in-memory processing for naive scripts.

### Problem
Ensure predictable memory profile and restart-safe progression.

### Decision
Process data page-by-page with cursor-based progression and per-batch commit.

### Why This Approach
- bounded memory behavior
- deterministic forward progress
- natural fit for resumability

### Tradeoffs
- additional control-flow complexity
- partial-batch replay risk under interruption before checkpoint commit

### Future Improvements
- partition-aware extraction and checkpointing
- record-level idempotency guarantees for strict replay safety
