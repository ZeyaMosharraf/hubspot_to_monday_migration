# HubSpot → Monday Migration Engine

Stateful CRM migration execution engine for moving HubSpot REST records into Monday GraphQL boards with resumable progression, bounded concurrency, and strict mapping integrity.

---

## Executive Summary

This repository implements a restart-safe migration pipeline for API-to-API CRM transfer. It is designed for operational migration runs, not ad hoc export scripts.

The engine advances incrementally through HubSpot cursor pagination, transforms source objects into Monday column payloads, executes bounded parallel load operations, and persists progress checkpoints to support deterministic recovery.

## Business Problem

CRM re-platforming from HubSpot to Monday introduces high operational risk:

- interrupted long-running migrations
- API throughput constraints and rate pressure
- source/target schema mismatch
- manual replay overhead after failures
- silent data-loss risk from weak validation

## Why This System Exists

This system exists to provide a controlled migration runtime with:

- resumable progression across large datasets
- explicit schema mapping control
- deterministic batch advancement
- fail-fast integrity behavior over silent partial success
- operationally simple execution model for one-time or phased migrations

## Architecture Overview

```text
Orchestrator (main.py)
  -> Extractor (clients/hubspot_client.py)
  -> Transformer (transformers/company_mapper.py)
  -> Loader (clients/monday_client.py)
  -> Checkpoint State (state/checkpoint.py + state/checkpoint.json)
  -> Runtime Config (config/settings.py, config/*_columns.py)
```

## Engineering Highlights

- Cursor-based incremental extraction
- Restart-safe checkpoint recovery
- Bounded worker pool for network-bound parallel loading
- Mapping abstraction between source properties and target columns
- Strict validation and explicit failure signaling
- Transform layer isolation for field normalization

## System Design Decisions

| Decision | Rationale | Consequence |
|---|---|---|
| Cursor pagination | Bounded memory and deterministic progression | Single forward traversal model |
| ThreadPoolExecutor (bounded) | Practical throughput for I/O-bound loads | Needs careful tuning for API limits |
| Fail-fast load behavior | Protect migration integrity | Single bad record can halt run |
| File-backed checkpoint state | Operational simplicity | Single-runner state model |
| Separate transform layer | Isolated schema evolution | Additional code surface to maintain |

## Data Flow

1. Read runtime settings and last checkpoint cursor.
2. Fetch one HubSpot page using `after` cursor + selected properties.
3. Transform each record into Monday payload structure.
4. Execute parallel load operations with bounded worker count.
5. Persist next cursor only after successful batch completion.
6. Repeat until no next cursor exists.

## REST → GraphQL Migration Flow

- **Source:** HubSpot CRM v3 REST objects (`/crm/v3/objects/companies`)
- **Target:** Monday GraphQL `create_item` mutation
- **Translation:** source properties -> normalized transform -> mapped Monday column IDs
- **Type handling:** date normalization and URL column payload shaping

## Checkpoint Recovery

- Checkpoint stores latest successful `after` cursor.
- Restart resumes from persisted cursor, not from zero.
- Cursor write occurs after batch execution completes.
- Recovery guarantees forward progress at batch granularity.

## Reliability Engineering

- Missing critical configuration fails at startup/runtime boundary.
- Missing mapping keys raise explicit errors.
- GraphQL API errors are surfaced and stop progression.
- Checkpoint write timing preserves deterministic batch boundaries.

## Scalability Considerations

- Bounded-memory extraction via paged cursor traversal
- Parallel I/O load path with controlled workers
- Throughput constrained by Monday API limits, latency, and mutation cost
- Current model is single-process, single-checkpoint-writer

## Performance Optimization

- Thread-based parallelism improves network-bound load throughput.
- Configurable page limit controls extraction granularity.
- Incremental progression reduces restart overhead for long migrations.

## Tradeoffs & Constraints

- Threaded concurrency chosen over async runtime complexity.
- Fail-fast integrity favors correctness over best-effort completion.
- Checkpoint file state favors simplicity over distributed scaling.
- Current loader path uses create mutation flow and should evolve to explicit idempotent upsert semantics for strict replay safety.

## Project Structure

```text
hubspot_to_monday_migration/
├── clients/
│   ├── hubspot_client.py
│   └── monday_client.py
├── config/
│   ├── settings.py
│   ├── hubspot_columns.py
│   └── monday_columns.py
├── state/
│   ├── checkpoint.py
│   └── checkpoint.json
├── transformers/
│   └── company_mapper.py
├── docs/
├── diagrams/
├── main.py
└── README.md
```

## Setup Instructions

1. Create and activate a Python virtual environment.
2. Install dependencies.
3. Configure `.env` with API credentials and runtime settings.
4. Confirm Monday board columns match `config/monday_columns.py`.

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

## Environment Variables

| Variable | Required | Purpose |
|---|---|---|
| `HUBSPOT_ACCESS_TOKEN` | Yes | HubSpot private app access token |
| `HUBSPOT_PAGE_LIMIT` | No (default `100`) | HubSpot page size per extraction batch |
| `MONDAY_API_TOKEN` | Yes | Monday API token |
| `MONDAY_COMPANY_BOARD_ID` | Yes | Monday board destination identifier |

Example:

```ini
HUBSPOT_ACCESS_TOKEN=...
HUBSPOT_PAGE_LIMIT=100
MONDAY_API_TOKEN=...
MONDAY_COMPANY_BOARD_ID=1234567890
```

## Running the Pipeline

```powershell
python main.py
```

Expected runtime behavior:

- prints migration start cursor
- processes batches until source exhaustion
- writes checkpoint after each successful batch
- emits summary with total processed and throughput

## Recovery Behavior

- **Interruption before checkpoint write:** current batch may be replayed on restart.
- **Interruption after checkpoint write:** next run resumes after committed cursor.
- **Manual restart:** simply re-run `python main.py`.
- **Fresh migration:** clear `state/checkpoint.json` cursor value.

## Future Roadmap

- idempotent upsert semantics for strict dedupe and replay safety
- retry/backoff policy for transient API failures
- centralized checkpoint state (Redis/Postgres)
- orchestrated scheduling and run observability
- DLQ + reconciliation reporting

## Documentation Links

- [Architecture](docs/architecture.md)
- [Checkpoint System](docs/checkpoint-system.md)
- [Engineering Decisions (ADR)](docs/engineering-decisions.md)
- [Scalability](docs/scalability.md)
- [Reliability](docs/reliability.md)
- [API Flow](docs/api-flow.md)
- [Performance](docs/performance.md)
- [Roadmap](docs/roadmap.md)
- [Diagrams](diagrams/)
