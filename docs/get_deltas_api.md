# GET /api/deltas

Endpoint: `GET /api/deltas?date_start={YYYY-MM-DD}&date_end={YYYY-MM-DD}`  
Controller: [`lib/controller/deltas_controller.rb`](../lib/controller/deltas_controller.rb)

---

> **Warning:** This endpoint returns all matching records in a single unbounded response. Querying large date ranges — particularly ones spanning many months or covering periods of high certificate activity — can produce very large payloads and place significant load on the service. Callers should keep date ranges as narrow as practical and be prepared to handle slow or large responses.

## Purpose

Returns a list of audit log events for EPC certificates that changed status within a given date range. This allows consumers to identify certificates that have been **removed** (opted out or cancelled) or had their **UPRN updated** since a given date, without having to re-fetch the entire dataset.

---

## Authentication

All requests require a valid Bearer token in the `Authorization` header.

---

## Parameters

Both parameters are **required**.

| Parameter    | Type   | Format     | Description                              |
|--------------|--------|------------|------------------------------------------|
| `date_start` | string | YYYY-MM-DD | Start of the date range (inclusive)      |
| `date_end`   | string | YYYY-MM-DD | End of the date range (inclusive)        |

A single-day range is supported by passing identical values for both parameters.

---

## End-to-End Flow

```
GET /api/deltas?date_start=2025-02-01&date_end=2025-02-28
  └── DeltasController
        └── UseCase::FetchAuditLogs#execute
              └── Boundary validation (nil dates, inverted range, includes today)
                    └── Gateway::AuditLogsGateway#fetch_logs
                          └── PostgreSQL: SELECT … FROM audit_logs WHERE timestamp BETWEEN …
                                └── BaseController#json_api_response (camelCase keys)
                                      └── { data: [...] }.to_json → HTTP 200
```

---

## Key Source Files

| File | Role |
|---|---|
| [lib/controller/deltas_controller.rb](../lib/controller/deltas_controller.rb) | Route definition; parameter schema validation; error handling |
| [lib/use_case/fetch_audit_logs.rb](../lib/use_case/fetch_audit_logs.rb) | Date validation logic; raises typed boundary errors |
| [lib/gateway/audit_logs_gateway.rb](../lib/gateway/audit_logs_gateway.rb) | SQL query against the `audit_logs` table; event type mapping |
| [db/migrate/20250217105543_create_audit_logs.rb](../db/migrate/20250217105543_create_audit_logs.rb) | Table creation |
| [db/migrate/20250219102223_add_constraint_audit_logs.rb](../db/migrate/20250219102223_add_constraint_audit_logs.rb) | Adds unique constraint on `(assessment_id, event_type)` |
| [db/migrate/20250730111836_add_timestamp_index_to_audit_log_table.rb](../db/migrate/20250730111836_add_timestamp_index_to_audit_log_table.rb) | Adds index on `timestamp` |

---

## Response Shape

Keys are camelCased. The response always wraps results in a top-level `data` array.

```json
{
  "data": [
    {
      "certificateNumber": "0000-0000-0000-0000-0001",
      "eventType": "removed",
      "timestamp": "2025-02-01T00:00:01.000Z"
    },
    {
      "certificateNumber": "0000-0000-0000-0000-0002",
      "eventType": "uprn_updated",
      "timestamp": "2025-02-03T00:00:01.000Z"
    }
  ]
}
```

---

## Event Types

The internal `event_type` values stored in the database are translated before being returned. Only two event types are exposed:

| Exposed `eventType` | Internal `event_type`(s)  | Meaning                                                    |
|---------------------|---------------------------|------------------------------------------------------------|
| `removed`           | `cancelled`, `opt_out`    | Certificate is no longer publicly available                |
| `uprn_updated`      | `address_id_updated`      | The UPRN associated with the certificate has been updated  |

`opt_in` events (a previously opted-out certificate becoming public again) are **written to the database** but **excluded from this endpoint's SQL query**. They are not surfaced in any response.

---

## Validation and Error Responses

| Condition | HTTP status | Error message |
|---|---|---|
| `date_start` or `date_end` is missing | `400` | `The search query was invalid - please provide a valid date range` |
| `date_start` is after `date_end` | `400` | `The search query was invalid - please provide a valid date range` |
| Date range includes today (or the future) | `400` | `The search query was invalid - the date cannot include today` |
| No audit log records found for the range | `404` | `No audit logs could be found for that query` |
| Unhandled exception | `500` | `{ "errors": [{ "code": "<message>" }] }` |

---

## Data Availability and Limitations

### How far back does data go?

The `audit_logs` table was created by migration `20250217105543` (17 February 2025). A subsequent migration on **19 February 2025** (`20250219102223`) added the unique constraint `(assessment_id, event_type)` and **truncated the table** as part of that change. Any events that occurred before 19 February 2025 were lost in that truncation. In practice, reliable audit log data begins from approximately **19 February 2025**.

### One record per certificate per event type

The table enforces a unique constraint on `(assessment_id, event_type)`. On conflict, the existing row is **updated in place** with the new timestamp (`ON CONFLICT … DO UPDATE SET timestamp=$3`). This means:

- Only the **most recent** occurrence of each event type per certificate is retained.
- Historical timestamps for repeated events (e.g. a certificate that was opted out, opted back in, and opted out again) are **overwritten** — only the latest timestamp survives.
- As a consequence, querying a wide date range does not guarantee you will see every occurrence of an event; you will only see events whose current timestamp falls within the range.

### No pagination

The endpoint has no `page_size` or `current_page` parameters. All records matching the date range are returned in a **single response**. There is no enforced limit on the number of records returned. Very large date ranges (e.g. spanning many months or covering a period of high activity) can produce large response payloads.

> **Load warning:** Because there is no pagination and no row cap, a wide date range can cause the database to return a very large result set in a single query. This increases memory pressure on the application server, serialisation time, and network transfer size — all of which can degrade response times for concurrent callers. To reduce load on the service, callers should query the narrowest date range that meets their needs (e.g. day-by-day or week-by-week) rather than requesting months of data in one call.

### Date range cannot include today

The use case explicitly rejects any date range where either bound equals today. Only historical data (up to and including yesterday) can be queried.

### No SQL-level cap on rows returned

The `fetch_logs` SQL query has no `LIMIT` clause. The only practical upper bound is the number of distinct `(assessment_id, event_type)` combinations whose `timestamp` falls within the requested range.

> **Load warning:** Without a `LIMIT`, a single request can cause a full sequential scan of all matching rows, potentially returning thousands of records. This places load directly on the database as well as the application layer and the caller.

---

## How Events Are Written

Audit log entries are inserted by three use cases:

| Use case | File | Event written |
|---|---|---|
| `OptOutCertificates` | [lib/use_case/opt_out_certificates.rb](../lib/use_case/opt_out_certificates.rb) | `opt_out` or `opt_in` |
| `CancelCertificates` | [lib/use_case/cancel_certificates.rb](../lib/use_case/cancel_certificates.rb) | `cancelled` |
| `UpdateCertificateAddresses` | [lib/use_case/update_certificate_addresses.rb](../lib/use_case/update_certificate_addresses.rb) | `address_id_updated` |
