# API Flow

## 1. End-to-End API Lifecycle

The migration runtime executes a repeating lifecycle:

1. HubSpot extraction request (REST)
2. Source page decode + cursor capture
3. Record transform to target contract
4. Monday mutation request (GraphQL) per record
5. Batch completion + checkpoint advancement

## 2. HubSpot Extraction Lifecycle

### Request

- **Endpoint:** `GET /crm/v3/objects/companies`
- **Headers:** `Authorization: Bearer <token>`
- **Query params:** `limit`, `properties`, optional `after`

### Response Handling

- `results[]` consumed as current batch
- `paging.next.after` consumed as next cursor

### Integration Constraints

- token validity
- API response latency/timeouts
- page traversal sequencing

## 3. Pagination Lifecycle

Pagination is cursor-based and forward-only:

- initial call with `after = null`
- each response provides next cursor
- loop terminates when next cursor is missing

Operational effect: deterministic incremental progression and bounded memory footprint.

## 4. Transformation Lifecycle

`transform_company()` maps raw source shape to load contract:

- extracts required properties
- normalizes date (`createdate`) to `YYYY-MM-DD`
- builds target columns payload

This isolates schema translation from API transport logic.

## 5. Schema Mapping Lifecycle

Mapping is resolved in loader using `config/monday_columns.py`.

Flow:

1. iterate transformed `columns` keys
2. resolve target `column_id`
3. fail on missing mapping
4. emit typed payload for target API

Mapping integrity acts as a schema guardrail.

## 6. Date Normalization Lifecycle

Source timestamps are ISO-8601 values from HubSpot.

Normalization path:

- parse source datetime
- convert to date string (`YYYY-MM-DD`)
- assign to mapped Monday date column

On parsing failure, date is set to `None` and omitted from load payload.

## 7. Monday Mutation Lifecycle

### Request

- **Endpoint:** `POST /v2`
- **Headers:** Authorization token, API version, content type
- **Body:** GraphQL `create_item` mutation + `column_values` JSON string

### Response Handling

- raise on HTTP failure
- parse JSON response
- raise on GraphQL `errors` payload

### Integration Constraints

- GraphQL complexity/rate limits
- board column type requirements
- per-request network latency

## 8. Batch Execution Lifecycle

For each source page:

1. submit concurrent load tasks via bounded thread pool
2. block until all tasks complete
3. advance checkpoint if batch succeeded

This enforces checkpoint commits only after complete batch processing.

## 9. Operational Sequence

```text
Start -> Load settings -> Load checkpoint
  -> Fetch page (HubSpot)
  -> Parallel transform+load (Monday)
  -> Save next cursor checkpoint
  -> Repeat until no next cursor
  -> Emit summary
```

## 10. Request/Response Control Points

| Stage | Request | Response Control | Failure Policy |
|---|---|---|---|
| Extract | HubSpot REST GET | parse `results` + `next.after` | fail fast |
| Transform | in-process mapping | normalized payload | fail via exceptions |
| Load | Monday GraphQL POST | check HTTP + GraphQL errors | fail fast |
| Commit | local checkpoint write | persisted cursor | fail on I/O error |
