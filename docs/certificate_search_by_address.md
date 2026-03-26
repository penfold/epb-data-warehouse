# Certificate Search by Address

This document explains how the `/api/domestic/search`, `/api/non-domestic/search`, and `/api/display/search` endpoints support searching for EPC certificates by address, covering the full request lifecycle from the API layer through to the database query.

---

## Endpoints

Three certificate search endpoints share the same address-search behaviour via a shared controller method:

| Endpoint | Assessment Types |
|---|---|
| `GET /api/domestic/search` | `SAP`, `RdSAP` |
| `GET /api/non-domestic/search` | `CEPC`, `CEPC-RR`, `DEC`, `DEC-RR`, `AC-CERT`, `AC-REPORT` |
| `GET /api/display/search` | `DEC`, `DEC-RR` |

All three delegate to `Controller::ApiBaseController#get_search_result`.

---

## Request Parameters

The `address` query parameter accepts a **partial or full address string**:

```
GET /api/domestic/search?address=9+new+union
```

It can be combined with other filters (date range, postcode, council, constituency, efficiency rating, UPRN). At least one search parameter must be present; if none are provided the request fails with a `400` error.

---

## Request Flow

```
HTTP Request
    │
    ▼
Controller::DomesticController (or CommercialController / DisplayController)
    │  calls get_search_result(assessment_type: [...])
    ▼
Controller::ApiBaseController#get_search_result
    │  1. Normalises pagination params (current_page, page_size)
    │  2. Validates JSON schema (SEARCH_SCHEMA) — ensures at least one parameter present
    │  3. Calls Helper::SearchParams.validate to validate dates, postcode, council,
    │     constituency, UPRN, and page size
    │  4. Runs UseCase::GetPagination to get total record count and pagination metadata
    │  5. Runs UseCase::AssessmentSearch to fetch the result page
    │
    ▼
UseCase::AssessmentSearch#execute
    │  Passes all parameters straight through to the gateway
    │  (removes efficiency_rating filter if all bands A–G are selected)
    │
    ▼
Gateway::AssessmentSearchGateway#fetch_assessments
    │  Builds and executes the SQL query (see below)
    ▼
PostgreSQL — assessment_search table
```

---

## The `assessment_search` Table

Certificates are indexed in a dedicated search table (`assessment_search`) populated when a certificate is lodged. The key columns are:

| Column | Type | Description |
|---|---|---|
| `assessment_id` | string | Certificate number (primary key component) |
| `address` | string (max 500) | Pre-computed, lowercased, concatenated address used for searching |
| `address_line_1–4` | string | Raw address fields returned in results |
| `post_town` | string | Post town |
| `postcode` | string | Formatted postcode |
| `council` | string | Local authority name (resolved from ONS postcode directory) |
| `constituency` | string | Westminster parliamentary constituency (resolved from ONS) |
| `uprn` | bigint | Unique Property Reference Number |
| `current_energy_efficiency_band` | string | e.g. `A`–`G` |
| `registration_date` | datetime | Certificate registration date |
| `assessment_type` | string | e.g. `SAP`, `RdSAP`, `CEPC` |
| `created_at` | datetime | Row insertion time |

---

## How the `address` Column Is Populated

When a certificate is inserted into `assessment_search`, the `address` column is computed by `Gateway::AssessmentSearchGateway#generate_address`:

```ruby
def generate_address(document:)
  arr = []
  keys = %i[address_line_1 address_line_2 address_line_3 address_line_4 post_town]
  keys.each do |key|
    arr.append(document[key]) unless document[key].nil?
  end
  arr.map(&:to_s).reject(&:empty?).join(" ").downcase
end
```

This concatenates all four address lines and the post town (skipping `nil` and empty values) with a single space separator, then **lowercases the entire string**.

**Example:** A certificate with:
- `address_line_1 = "1 Some Street"`
- `address_line_2 = nil`
- `post_town = "Whitbury"`

…produces `address = "1 some street whitbury"`.

---

## Address Matching Logic

### SQL `LIKE` with wildcard wrapping

The address search uses a **case-insensitive substring match** via a PostgreSQL `LIKE` clause:

```sql
AND address LIKE $N
```

Before binding the query parameter, the gateway wraps the user-supplied string in `%` wildcards **and lowercases it**:

```ruby
# in Gateway::AssessmentSearchGateway#get_bindings
arr << ActiveRecord::Relation::QueryAttribute.new(
  "address",
  "%#{this_args[:address].downcase}%",
  ActiveRecord::Type::String.new,
)
```

So a user input of `"9 New Union"` becomes the binding value `"%9 new union%"`, which matches any `address` value that contains that substring.

Because the `address` column is always stored in lowercase (see above), and the search term is also lowercased before binding, the match is effectively **case-insensitive** without requiring a `ILIKE` or `LOWER()` call.

### GIN trigram index

To make the `LIKE '%…%'` query efficient (leading-wildcard `LIKE` queries cannot use a normal B-tree index), the `address` column has a **GIN trigram index** created via the `pg_trgm` PostgreSQL extension:

```sql
CREATE EXTENSION pg_trgm;

CREATE INDEX index_assessment_search_on_address_trigram
ON assessment_search
USING gin (address gin_trgm_ops);
```

The PostgreSQL query planner uses this index to quickly find rows where the trigram set of the `address` value contains the trigrams of the search pattern, supporting fast substring searches across millions of records.

### Summary of matching behaviour

| User input | Stored as binding | Matches |
|---|---|---|
| `"9 new union"` | `"%9 new union%"` | Any address containing `9 new union` |
| `"2 Banana Street"` | `"%2 banana street%"` | Any address containing `2 banana street` |
| `"SW10"` | `"%sw10%"` | Any address containing `sw10` (also matched by the `postcode` filter if exact postcode format is used) |

---

## Parameter Validation

The `address` parameter itself has **no server-side format validation** beyond the JSON schema confirming it is a string. It is bound as a parameterised query value (preventing SQL injection), and only limited by:

- The 500-character `address` column limit in the database.
- The requirement that at least one of the recognised search parameters be present.

---

## Pagination

Results are paginated using the `current_page` and `page_size` parameters (defaults: page 1, 5000 rows). `UseCase::GetPagination` first runs a `COUNT(*)` query with the same filters to determine `total_records` and `total_pages`. The main data query then adds `ORDER BY registration_date DESC`, `LIMIT`, and `OFFSET` clauses.

---

## Response Shape

```json
{
  "data": [
    {
      "certificateNumber": "0000-0000-0000-0000-0000",
      "addressLine1": "1 Some Street",
      "addressLine2": null,
      "addressLine3": null,
      "addressLine4": null,
      "postTown": "Whitbury",
      "postcode": "SW10 0AA",
      "council": "Hammersmith and Fulham",
      "constituency": "Chelsea and Fulham",
      "currentEnergyEfficiencyBand": "B",
      "registrationDate": "2020-05-04",
      "uprn": 100121241798
    }
  ],
  "pagination": {
    "totalRecords": 1,
    "currentPage": 1,
    "totalPages": 1,
    "nextPage": null,
    "prevPage": null,
    "pageSize": 5000
  }
}
```

---

## Key Source Files

| File | Role |
|---|---|
| [lib/controller/domestic_controller.rb](../lib/controller/domestic_controller.rb) | Route definition for `/api/domestic/search` |
| [lib/controller/api_base_controller.rb](../lib/controller/api_base_controller.rb) | `get_search_result` — parameter handling, validation, orchestration |
| [lib/helper/search_params.rb](../lib/helper/search_params.rb) | Input validation (dates, postcode, council, constituency, page size) |
| [lib/use_case/assessment_search.rb](../lib/use_case/assessment_search.rb) | Thin use case that delegates to the gateway |
| [lib/use_case/get_pagination.rb](../lib/use_case/get_pagination.rb) | Calculates pagination metadata via a COUNT query |
| [lib/gateway/assessment_search_gateway.rb](../lib/gateway/assessment_search_gateway.rb) | SQL query construction; `generate_address` and `get_bindings` methods |
| [db/migrate/20250630091828_create_assessment_search.rb](../db/migrate/20250630091828_create_assessment_search.rb) | Creates `assessment_search` table and GIN trigram index |
| [db/migrate/20241128095332_enable_trigram_ext.rb](../db/migrate/20241128095332_enable_trigram_ext.rb) | Enables the `pg_trgm` PostgreSQL extension |
