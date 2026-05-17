# Architecture

## 1. System Boundaries

This platform is a **single-process migration runtime** bounded by two external systems:

- **Source boundary:** HubSpot CRM REST API (`/crm/v3/objects/{object}`)
- **Target boundary:** Monday GraphQL API (`/v2`)

Internal responsibilities are strictly separated:

- Orchestration (`main.py`)
- Source extraction (`clients/hubspot_client.py`)
- Transformation (`transformers/company_mapper.py`)
- Target loading (`clients/monday_client.py`)
- Runtime/state configuration (`config/*`, `state/checkpoint.py`)

## 2. Component Topology

```text
main.py
  -> load_settings()
  -> load_checkpoint()
  -> fetch_object() [HubSpot]
  -> process_record() -> transform_company()
  -> upsert_item() [Monday]
  -> save_checkpoint()
```

## 3. Orchestration Flow

1. Load environment-driven runtime settings.
2. Load last successful cursor checkpoint.
3. Fetch source page with explicit property projection.
4. Execute per-record transform + load in bounded thread pool.
5. Persist next cursor after successful batch completion.
6. Repeat until source paging ends.

This creates deterministic, forward-only progression with restart continuity.

## 4. Extraction Flow

Extraction is cursor-driven and page-bounded:

- input: `object_type`, selected `properties`, `after`, `limit`
- output: `results`, `next_after`

Key boundary behavior:

- enforces token presence
- uses HubSpot pagination cursor (`after`)
- avoids loading full dataset into memory

## 5. Transform Lifecycle

Transform logic is isolated from API clients:

- receives raw HubSpot object
- normalizes fields into target contract (`item_name`, `columns`)
- performs date coercion to Monday-compatible format

Design benefit: schema evolution is localized to transform + mapping layers.

## 6. Loading Lifecycle

Loader constructs Monday GraphQL mutation payload:

- validates mapping key existence
- normalizes URL-style fields for typed column payload
- submits mutation with API versioned headers
- raises on HTTP or GraphQL error

Load behavior is intentionally strict to prevent silent data drift.

## 7. Checkpoint Lifecycle

Checkpoint state (`state/checkpoint.json`) tracks the latest successful source cursor.

- read at startup (`load_checkpoint`)
- write after batch success (`save_checkpoint`)
- used for resume after interruption

Checkpoint semantics are batch-level progress commits.

## 8. API Boundaries and Contracts

| Boundary | Contract | Failure Mode |
|---|---|---|
| HubSpot extract | REST JSON records + `paging.next.after` | request/credential/network error |
| Transform | mapped `item_data` dict | conversion/normalization mismatch |
| Monday load | GraphQL mutation with mapped column payload | HTTP error / GraphQL error payload |
| Checkpoint state | JSON `{"after": ...}` | file I/O/state corruption |

## 9. Dependency Relationships

- `main.py` depends on config, clients, transform, checkpoint modules.
- `hubspot_client.py` depends on `settings` and source property definitions.
- `monday_client.py` depends on `settings` and target column mapping.
- `transformers` stay API-agnostic and depend only on Python stdlib.

This keeps integration-specific behavior out of orchestration control flow.

## 10. Concurrency Strategy

The runtime uses `ThreadPoolExecutor(max_workers=6)` for load-side parallelism.

Why this strategy:

- workload is network-bound
- bounded workers control API pressure
- implementation complexity remains low

Current scaling envelope:

- single process
- single checkpoint writer
- concurrency limited by API latency and platform constraints

## 11. Architecture Reasoning

- Prefer **incremental deterministic progression** over in-memory bulk transfer.
- Prefer **strict failures** over hidden partial success.
- Prefer **simple state durability** over distributed coordination for current scope.

## 12. Tradeoffs

- File-backed checkpoint is operationally simple but not distributed-safe.
- Threading is practical but less precise than async backpressure control.
- Fail-fast behavior can stop runs on one bad record; this is deliberate integrity bias.

## 13. Scaling Implications

Short-term:

- tune page size and worker count against API constraints.

Medium-term:

- partition extraction space into independent shards.
- centralize checkpoint state to support multi-worker execution.

Long-term:

- adopt orchestrated, observable, idempotent migration runtime with replay controls.
